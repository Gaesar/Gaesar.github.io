---
title: MCP
author: Gaesar
date: 2026-10-06 23:21:48
tags: 从0学习Agent
categories: AI-Agent
---

# MCP

## MCP解决的问题

在使用Function Calling调用外部工具的时候，工具schema和工具的执行都需要手动写，当在另一个应用中调用相同的工具时，要重新写一遍工具schema和工具的执行，同一个工具发生变化时，所有的应用都要修改。

这个时候AI应用和工具是N*M的关系，每个应用都要自己实现一遍工具。

MCP是Anthropic制定的一套通信协议，方便AI应用对接外部工具。

AI应用实现MCP Host + MCP Client，工具提供方实现MCP Server，通过MCP协议，AI应用可以方便地接入任何工具，schema和工具的执行完全由工具提供方实现，AI应用只需要发送请求和转发返回给大模型就好了。

这个时候AI应用和工具是N+M的关系，每个工具一次开发，到处可用。

## MCP三层架构

MCP 是**Host-Client-Server**三层模型，基于 JSON-RPC 2.0 通信。

### 1.宿主Host

AI应用，调用 LLM。

### 2.MCP Client

在Host内部，一个 Client 对应一个 MCP Server，Host 可以同时启动多个 Client，连多个 MCP Server。

### 3.MCP Server

AI应用外部的独立程序（本地 Python/Node 进程，或远程 HTTP 服务），通过 MCP 协议对外暴露能力。

### 4.通信方式

通信传输方式两种：

- `stdio`：本地进程通信
- `Streamable HTTP`：远程网络访问 MCP Server

## 接入MCP前后链路对比

### 纯Function Calling链路

```
1. 用户提问
2. AI应用把：用户 query + 手写写死的工具`schema列表`发给 LLM
3. LLM 判断需要调用工具，返回 `tool_call`（工具名 + 参数）
4. AI应用内部硬编码，执行本地工具函数
5. 拿到函数返回结果，塞进消息上下文，再次发给 LLM
6. LLM 拿到结果，生成最终回答
```

### 接入MCP后的链路

```
一、启动阶段
1.AI应用（Host）启动 MCP Client，和 MCP Server 建立连接
2.MCP 握手，Host 向 Server 请求`list_tools`
3.MCP Server 返回它所有工具的JSON Schema（工具名称、描述、入参）
4.AI应用（Host） 把从 MCP Server 拿到的全部工具 Schema收集起来，打包，作为 Function Call 的工具列表传给 LLM
二、运行阶段
1. 用户提问
2. AI应用（Host） 把用户query + 从 MCP Server 拉取来的工具 schema一起发给 LLM
3. LLM 判断需要调用工具，返回 `tool_call`（工具名 + 参数）
4. AI应用（Host） 把这个 tool_call 转发给对应的 MCP Client，调用 MCP 协议的`tools/call` RPC 请求，发给 MCP Server 进程
5. MCP Server 收到请求，在它自己的独立进程里执行对应的工具逻辑（读文件、跑 SQL），返回结果给 AI应用（Host）
6. AI应用（Host） 拿到 MCP Server 返回的工具结果，把结果追加到对话上下文
7. AI应用（Host） 把完整上下文再次发给 LLM，LLM 结合工具输出生成回答
```

 ### MCP的作用

和大模型交互的没有变，还是AI应用，还是Function Calling。

变的是工具的定义和执行，不再自己写了，而是通过MCP调用外部服务。

### MCP额外提供的两个功能

Resources、Prompts**不是 LLM 通过 Function Call 调用**，是 Host 主动拉取

1.Resources（读取数据源）

```
场景：用户问 “分析这个文档”
1. Host 识别用户需要读取某个资源（文件 URI）
2. Host 通过 MCP Client 调用`resources/get`，从 MCP Server 拉取文档内容
3. Host 把文档内容直接塞进上下文，再发给 LLM。
```

2.Prompts（提示模板）

```
1. Host 向 MCP Server 调用`prompts/get`拉取预设模板（带参数渲染）
2. Host 拿到渲染后的 prompt，作为系统提示词丢进上下文给 LLM
```

prompt 是返回给拼接好的提示词，而 resources 是直接返回想要的资源文件。

## MCP细节

### 消息层

1.握手（initialize）

```
→ {"method": "initialize", "params": {
     "protocolVersion": "2025-06-18",
     "capabilities": {...},
     "clientInfo": {"name": "miniagent", "version": "0.6"}}}
← {"result": {
     "protocolVersion": "2025-06-18",
     "capabilities": {"tools": {}},
     "serverInfo": {"name": "demo-server"}}}
```

`capabilities` 声明各自支持哪些能力类别（tools/resources/prompts），没声明的能力后面不许用。

2.发现（tools/list）

```
→ {"method": "tools/list"}
← {"result": {"tools": [
     {"name": "current_time",
      "description": "返回当前的日期和时间",
      "inputSchema": {"type": "object", "properties": {}}},
     {"name": "project_stats",
      "description": "统计一个目录下的 Python 文件数量和总行数",
      "inputSchema": {"type": "object",
                      "properties": {"directory": {"type": "string"}},
                      "required": ["directory"]}}]}}
```

拉去工具的schema列表，`inputSchema` 就是 JSON Schema，和 Function Calling 的工具 Schema 是同一套语言，MCP 工具进 Agent 的工具表时，无需格式转换。

3.调用（tools/call）

```
→ {"method": "tools/call", "params": {
     "name": "current_time", "arguments": {}}}
← {"result": {"content": [
     {"type": "text", "text": "2026-07-05 16:17:40"}],
     "isError": false}}
```

返回的 `content` 是块数组，除了 text 还可以是图片等类型

`isError` 标记执行失败。注意失败也走正常返回而非协议异常，因为对模型来说，失败也是有用信息。

### 传输层

通信传输方式两种：

- `stdio`：本地进程通信，用于本地工具：文件系统、git、本地数据库
- `Streamable HTTP`：远程网络访问 MCP Server，适合云端/共享场景：公司内部 API 网关、SaaS 工具服务

MCP over stdio 的约定：`stdout` 专门用来传输 JSON-RPC 协议报文（唯一信道），`print()` 默认输出到 `stdout`，一旦在 server 代码打印调试文本，就会污染 JSON-RPC 报文流，MCP Client 读 stdout 时，读到一堆混杂的垃圾文本，JSON 解析报错。

stdio底层原理

```
1. Host 调用 `subprocess` 拉起 Server 作为独立子进程
2. 父进程 (Host/MCP Client) 把子进程的 stdout 接管，作为MCP 协议的单向输出通道
   - Server 所有发给 Client 的 MCP JSON-RPC 消息，必须输出到 stdout
   - 格式：一行一条完整 JSON，以换行分隔（`\n`作为消息分隔符）
3. `stdin`：反向通道，Client 把 JSON-RPC 请求发给 Server（子进程读 stdin）
```

调试信息必须走 stderr 或日志文件。

## 多Server

### 命名冲突

多个Server工具名称相同，或者和本地工具名称相同。

使用前缀路由：工具进表时改名为 `mcp__servername__toolname`，执行时按前缀拆回去。

### 工具爆炸

上百个工具全部发送给模型，模型选择的准确率下降。

按需加载，根据任务先筛一轮相关工具再进工具表。

## MCP简单示例代码

### MCP Server

```python
import time
from pathlib import Path

from mcp.server.mcpserver import MCPServer

server = MCPServer("demo-server")

@server.tool()
def project_stats(directory: str) -> str:
    """统计一个目录下的 Python 文件数量和总行数"""
    root = Path(directory)
    if not root.is_dir():
        return f"错误：目录不存在 {directory}"
    files = [p for p in root.rglob("*.py") if p.is_file()]
    lines = sum(len(p.read_text(encoding="utf-8").splitlines()) for p in files)
    return f"{directory}：{len(files)} 个 Python 文件，共 {lines} 行"
```

`"""统计一个目录下的 Python 文件数量和总行数"""`MCP 拿它当做 `description` 字段，给大模型看这个工具能干啥。

`def project_stats(directory: str) -> str:`自动生成参数 JSON Schema：

```json
{
  "type": "object",
  "properties": {
    "directory": {
      "type": "string"
    }
  },
  "required": ["directory"]
}

```

`project_stats` 函数名作为工具名

### MCP Client

```python
import sys
from pathlib import Path

from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

SERVER = StdioServerParameters(
    command=sys.executable,
    args=[str(Path(__file__).with_name("mcp_server.py"))],
)
MCP_PREFIX = "mcp__"   # 路由标记：带这个前缀的工具走 MCP

async def _list_tools() -> list[dict]:
    async with stdio_client(SERVER) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()          # 握手
            tools = await session.list_tools()  # 发现
    return [
        {
            "type": "function",
            "function": {
                "name": MCP_PREFIX + t.name,
                "description": t.description or "",
                "parameters": t.input_schema,    # MCP Schema 直接就是 FC Schema
            },
        }
        for t in tools.tools
    ]

async def _call_tool(name: str, args: dict) -> str:
    async with stdio_client(SERVER) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            result = await session.call_tool(name, args)
    text = "\n".join(b.text for b in result.content if b.type == "text")
    return f"错误：MCP 工具执行失败：{text}" if result.is_error else text
```

mcp client向mcp server发出工具执行请求，mcp server通过@server.tool装饰器维护了一个函数名到函数的映射字典，受到请求后执行对应的函数。

## 实验

1.加入了两个本地mcp，提供了两个工具，获取当前时间和统计代码。

```
你> 用工具查一下现在几点，再统计./code_data目录下 Python 代码规模，一起汇报。
这是第 1 次循环...
MiniAgent observation ...
MiniAgent Reasoning > The user wants me to check the current time and count Python code in ./code_data directory. Let me make both calls in parallel since they're independent.

MiniAgent Acting >mcp__current_time的执行结果：2026-10-07 15:23:18
MiniAgent Acting >mcp__project_stats的执行结果：./code_data：5 个 Python 文件，共 67 行
这是第 2 次循环...
MiniAgent observation ...
MiniAgent> 

已查到两项结果：

**当前时间**：2026-10-07 15:23:18

**`./code_data` 代码规模**：5 个 Python 文件，共 67 行（平均每个文件约 13 行，属于很小的代码集）。

两项查询相互独立，已并行发起并一次性拿到结果，未对任何文件做修改。
```

2.在MCP Server添加了新的函数，转换日期格式，体验了“server加工具，client0改动”

```
你> 用工具查一下现在几点，给我年月日时分秒的格式。
这是第 1 次循环...
MiniAgent observation ...
MiniAgent Reasoning > The user is asking for the current time in a specific format that includes year, month, day, hour, minute, and second. I need to call the current_time tool first, then use the time_format tool to convert it to the desired format. Since the second call depends on the first, I'll need to execute them sequentially rather than in parallel.

MiniAgent Acting >mcp__current_time的执行结果：2026-10-07 16:11:33
这是第 2 次循环...
MiniAgent observation ...
MiniAgent Reasoning > Now I'm applying the time formatting to this.

MiniAgent Acting >mcp__time_format的执行结果：2026年10月07日 16时11分33秒
这是第 3 次循环...
MiniAgent observation ...
MiniAgent> 

现在的时间是：**2026年10月07日 16时11分33秒**
```

3.安装了搜索引擎MCP，测试了联网搜索功能。

```
你> 使用网页查询 MCP 工具，打开 Box Office Mojo，查询出2026 年度全球累计票房榜前5名的名称、上映时间、票房。只使用 MCP 搜索和 fetch，禁止 bash/curl。
这是第 1 次循环...
MiniAgent observation ...
MiniAgent Reasoning > 用户要求使用网页查询 MCP 工具，查询 2026 年度全球累计票房榜前 5 名。禁止使用 bash/curl。

我需要先搜索 Box Office Mojo 的 2026 年度全球票房榜页面，然后读取该页面。

让我先用 web_search 搜索。

[10/07/26 19:49:31] INFO     Processing request of type           server.py:733
                             CallToolRequest                                   
                    INFO     QSL: enhanced query with categories: server.py:278
                             ['dev_tools']                                     
[10/07/26 19:49:32] INFO     HTTP Request: HEAD                 _client.py:1740
                             https://www.baidu.com "HTTP/1.1                   
                             200 OK"                                           
                    INFO     HTTP Request: HEAD                 _client.py:1740
                             https://www.bing.com "HTTP/1.1 302                
                             Found"                                            
[10/07/26 19:49:33] INFO     HTTP Request: HEAD                 _client.py:1740
                             https://yandex.com "HTTP/1.1 200                  
                             Ok"                                               
[10/07/26 19:49:34] INFO     HTTP Request: HEAD                 _client.py:1740
                             https://cn.bing.com/ "HTTP/1.1 200                
                             OK"                                               
                    INFO     HTTP Request: GET                  _client.py:1740
                             https://cn.bing.com/?ensearch=1                   
                             "HTTP/1.1 200 OK"                                 
                    INFO     HTTP Request: GET                  _client.py:1740
                             https://www.bing.com/search?q=Box+                
                             Office+Mojo+2026+yearly+worldwide+                
                             box+office+github+docker+kubernete                
                             s&count=10&ensearch=1&FORM=BESBTB                 
                             "HTTP/1.1 302 Found"                              
                    INFO     HTTP Request: GET                  _client.py:1740
                             https://cn.bing.com/search?q=Box+O                
                             ffice+Mojo+2026+yearly+worldwide+b                
                             ox+office+github+docker+kubernetes                
                             &count=10&ensearch=1&FORM=BESBTB                  
                             "HTTP/1.1 200 OK"                                 
                    INFO     Processing request of type           server.py:733
                             ListToolsRequest                                  
MiniAgent Acting >mcp__web_search的执行结果：## "Box Office Mojo 2026 yearly worldwide box office github docker kubernetes" — 8 results via native:bing_intl (375ms, cache:MISS)

1. **Box | Login** — account.box.com
   https://account.box.com/login
   Sign in privately. Learn more. Don't have an account? Sign Up.

2. **Box | AI-Powered Content Management, Workflow & Collaboration** — box.com
   https://www.box.com/
   By investing in a cloud content management platform like Box and leveraging other best-of-breed technology partners, we have been …

3. **Box | Login** — account.box.com
   https://account.box.com/login/2fa
   Your Admin has enabled 2-step verification for your account. When logging in from a new browser or device, we will send you a text …

4. **Box | Login** — app.box.com
   https://app.box.com/login/popup
   Sign In to Your Account Email Address Password Log In

5. **Download Box Drive | Stream Files & Replace Network File Shares** — box.com
   https://www.box.com/drive
   Box Drive is the incredibly simple way to work with all of your files — even billions of files — right from your desktop, taking up very …

6. **Box.org** — box.org
   https://www.box.org/home
   Box gives you a single, secure platform to manage the entire content lifecycle, from file creation and sharing to e-signature and …

7. **Box | Login** — my.account.box.com
   https://my.account.box.com/login/reset
   Enter your email to get password reset instructions sent to you.

8. **Get started | Box** — box.org
   https://www.box.org/use-box
   With donated or discounted access, you get a single place to collaborate, manage, and secure all your content, simplify your …

[10/07/26 19:49:38] INFO     Processing request of type           server.py:733
                             CallToolRequest                                   
                    INFO     QSL: enhanced query with categories: server.py:278
                             ['dev_tools']                                     
                    INFO     HTTP Request: HEAD                 _client.py:1740
                             https://www.baidu.com "HTTP/1.1                   
                             200 OK"                                           
                    INFO     HTTP Request: HEAD                 _client.py:1740
                             https://www.bing.com "HTTP/1.1 302                
                             Found"                                            
                    INFO     HTTP Request: HEAD                 _client.py:1740
                             https://cn.bing.com/ "HTTP/1.1 200                
                             OK"                                               
[10/07/26 19:49:39] INFO     HTTP Request: HEAD                 _client.py:1740
                             https://yandex.com "HTTP/1.1 200                  
                             Ok"                                               
[10/07/26 19:49:41] INFO     HTTP Request: GET                  _client.py:1740
                             https://cn.bing.com/?ensearch=1                   
                             "HTTP/1.1 200 OK"                                 
                    INFO     HTTP Request: GET                  _client.py:1740
                             https://www.bing.com/search?q=boxo                
                             fficemojo.com+year+2026+worldwide+                
                             box+office+chart+github+docker+kub                
                             ernetes&count=10&ensearch=1&FORM=B                
                             ESBTB "HTTP/1.1 302 Found"                        
                    INFO     HTTP Request: GET                  _client.py:1740
                             https://cn.bing.com/search?q=boxof                
                             ficemojo.com+year+2026+worldwide+b                
                             ox+office+chart+github+docker+kube                
                             rnetes&count=10&ensearch=1&FORM=BE                
                             SBTB "HTTP/1.1 200 OK"                            
                    INFO     Processing request of type           server.py:733
                             ListToolsRequest                                  
MiniAgent Acting >mcp__web_search的执行结果：## "boxofficemojo.com year 2026 worldwide box office chart github docker kubernetes" — 5 results via native:bing_intl (375ms, cache:MISS)

1. **Home - Box Office Mojo** — boxofficemojo.com
   https://www.boxofficemojo.com/
   11 hours ago · Box Office Mojo and IMDb are trademarks or registered trademarks of IMDb.com, Inc. or its affiliates. Conditions of …

2. **Home - Box Office Mojo** — boxofficemojo.com
   https://www.boxofficemojo.com/?sortDir=asc
   11 hours ago · Track box office performance, movie rankings, and industry trends on Box Office Mojo.

3. **Domestic Yearly Box Office - Box Office Mojo** — bing.com
   https://www.bing.com/ck/a?!&&p=632da4846edf130fcb22e581fddf5c0a687dff76bad46428e97783e46e169e21JmltdHM9MTc5MTMzMTIwMA&ptn=3&ver=2&hsh=4&fclid=2e6a4b16-5a60-6827-08e5-5cfd5b066946&u=a1aHR0cHM6Ly93d3cuYm94b2ZmaWNlbW9qby5jb20veWVhci8_c29ydERpcj1hc2M&ntb=1
   Data as of Oct 6, 2:48 PDT Year

4. **Daily Domestic Box Office Monday, October 5, 2026 - The Numbers** — the-numbers.com
   https://www.the-numbers.com/daily-box-office-chart
   3 days ago · The Numbers provides detailed movie financial analysis, including box office, DVD and Blu-ray sales reports, and …

5. **Top box office (US) - IMDb** — imdb.com
   https://www.imdb.com/chart/boxoffice/
   Weekend Gross: $2.5M Total Gross: $955M Weeks Released: 10 8.0 (322K) Rate Mark as watched Reported by Box Office Mojo © …

这是第 2 次循环...
MiniAgent observation ...
MiniAgent Reasoning > The search suggests current date is around Oct 2026. Let's fetch the Box Office Mojo yearly worldwide page: https://www.boxofficemojo.com/year/world/ or /year/world/2026/.

Let me try mcp__web_read on https://www.boxofficemojo.com/year/world/2026/.

[10/07/26 19:51:44] INFO     Processing request of type           server.py:733
                             CallToolRequest                                   
[10/07/26 19:51:47] INFO     HTTP Request: GET                  _client.py:1740
                             https://www.boxofficemojo.com/year                
                             /world/2026/ "HTTP/1.1 200 OK"                    
                    INFO     Processing request of type           server.py:733
                             ListToolsRequest                                  
MiniAgent Acting >mcp__web_read的执行结果：## 2026 Worldwide Box Office - Box Office Mojo

> URL: https://www.boxofficemojo.com/year/world/2026/ | Method: trafilatura | Quality: 58% (density: 92%, structure: 0%, clean: 100%, complete: 0%) | cache:MISS

| [Rank](https://www.boxofficemojo.com?sort=rank&ref_=bo_ydw__resort#table) | Release Group | [Worldwide](https://www.boxofficemojo.com?sort=worldwideGrossToDate&ref_=bo_ydw__resort#table) | [Domestic](https://www.boxofficemojo.com?sort=domesticGrossToDate&ref_=bo_ydw__resort#table) | [%](https://www.boxofficemojo.com?sort=domesticGrossToDatePercent&ref_=bo_ydw__resort#table) | [Foreign](https://www.boxofficemojo.com?sort=foreignGrossToDate&ref_=bo_ydw__resort#table) | [%](https://www.boxofficemojo.com?sort=foreignGrossToDatePercent&ref_=bo_ydw__resort#table) | 
|---|---|---|---|---|---|---|
| 1 | [Spider-Man: Brand New Day](https://www.boxofficemojo.com/releasegroup/gr3277935365/?ref_=bo_ydw_table_1) | $2,506,251,053 | $954,734,627 | 38.1% | $1,551,516,426 | 61.9% | 
| 2 | [The Odyssey](https://www.boxofficemojo.com/releasegroup/gr359027461/?ref_=bo_ydw_table_2) | $1,768,084,090 | $620,585,090 | 35.1% | $1,147,499,000 | 64.9% | 
| 3 | [Toy Story 5](https://www.boxofficemojo.com/releasegroup/gr594236165/?ref_=bo_ydw_table_3) | $1,148,032,461 | $480,294,461 | 41.8% | $667,738,000 | 58.2% | 
| 4 | [Michael](https://www.boxofficemojo.com/releasegroup/gr3549843973/?ref_=bo_ydw_table_4) | $1,028,703,388 | $372,303,388 | 36.2% | $656,400,000 | 63.8% | 
| 5 | [The Super Mario Galaxy Movie](https://www.boxofficemojo.com/releasegroup/gr513036805/?ref_=bo_ydw_table_5) | $1,012,578,077 | $429,814,610 | 42.4% | $582,763,467 | 57.6% | 
| 6 | [The Devil Wears Prada 2](https://www.boxofficemojo.com/releasegroup/gr3934212869/?ref_=bo_ydw_table_6) | $693,255,651 | $220,556,651 | 31.8% | $472,699,000 | 68.2% | 
| 7 | [Project Hail Mary](https://www.boxofficemojo.com/releasegroup/gr2625458949/?ref_=bo_ydw_table_7) | $684,489,947 | $344,050,007 | 50.3% | $340,439,940 | 49.7% | 
| 8 | [Pegasus 3](https://www.boxofficemojo.com/releasegroup/gr3427357445/?ref_=bo_ydw_table_8) | $656,459,523 | $1,374,946 | 0.2% | $655,084,577 | 99.8% | 
| 9 | [Minions & Monsters](https://www.boxofficemojo.com/releasegroup/gr3362673413/?ref_=bo_ydw_table_9) | $525,512,021 | $183,171,260 | 34.9% | $342,340,761 | 65.1% | 
| 10 | [Obsession](https://www.boxofficemojo.com/releasegroup/gr4050998021/?ref_=bo_ydw_table_10) | $520,006,715 | $263,451,715 | 50.7% | $256,555,000 | 49.3% | 
| 11 | [Backrooms](https://www.boxofficemojo.com/releasegroup/gr1011503877/?ref_=bo_ydw_table_11) | $400,583,134 | $197,523,513 | 49.3% | $203,059,621 | 50.7% | 
| 12 | [Hoppers](https://www.boxofficemojo.com/releasegroup/gr3213185541/?ref_=bo_ydw_table_12) | $389,685,783 | $166,010,783 | 42.6% | $223,675,000 | 57.4% | 
| 13 | [Once Upon a Time in the Middle East](https://www.boxofficemojo.com/releasegroup/gr440554245/?ref_=bo_ydw_table_13) | $350,730,039 | - | - | $350,730,039 | 100% | 
| 14 | [Star Wars: The Mandalorian and Grogu](https://www.boxofficemojo.com/releasegroup/gr577458949/?ref_=bo_ydw_table_14) | $345,602,060 | $177,730,060 | 51.4% | $167,872,000 | 48.6% | 
| 15 | [Kung Fu Soccer](https://www.boxofficemojo.com/releasegroup/gr1816351493/?ref_=bo_ydw_table_15) | $325,523,144 | - | - | $325,523,144 | 100% | 
| 16 | [Moana](https://www.boxofficemojo.com/releasegroup/gr3146076677/?ref_=bo_ydw_table_16) | $323,645,663 | $126,229,663 | 39% | $197,416,000 | 61% | 
| 17 | [All Wishes Come True!](https://www.boxofficemojo.com/releasegroup/gr1833128709/?ref_=bo_ydw_table_17) | $303,382,702 | $713,708 | 0.2% | $302,668,994 | 99.8% | 
| 18 | [Dear You](https://www.boxofficemojo.com/releasegroup/gr1867404037/?ref_=bo_ydw_table_18) | $291,035,441 | $1,296,824 | 0.4% | $289,738,617 | 99.6% | 
| 19 | [Resident Evil](https://www.boxofficemojo.com/releasegroup/gr3598406405/?ref_=bo_ydw_table_19) | $247,641,828 | $126,237,519 | 51% | $121,404,309 | 49% | 
| 20 | [Disclosure Day](https://www.boxofficemojo.com/releasegroup/gr426332933/?ref_=bo_ydw_table_20) | $243,797,645 | $116,636,645 | 47.8% | $127,161,000 | 52.2% | 
| 21 | [Wuthering Heights](https://www.boxofficemojo.com/releasegroup/gr661082885/?ref_=bo_ydw_table_21) | $241,801,072 | $84,001,072 | 34.7% | $157,800,000 | 65.3% | 
| 22 | [Scary Movie](https://www.boxofficemojo.com/releasegroup/gr2657440517/?ref_=bo_ydw_table_22) | $231,886,052 | $108,277,762 | 46.7% | $123,608,290 | 53.3% | 
| 23 | [Blades of the Guardians](https://www.boxofficemojo.com/releasegroup/gr1850430213/?ref_=bo_ydw_table_23) | $215,363,913 | $1,606,672 | 0.7% | $213,757,241 | 99.3% | 
| 24 | [Scream 7](https://www.boxofficemojo.com/releasegroup/gr895701765/?ref_=bo_ydw_table_24) | $207,999,405 | $121,935,967 | 58.6% | $86,063,438 | 41.4% | 
| 25 | [Scare Out](https://www.boxofficemojo.com/releasegroup/gr105534213/?ref_=bo_ydw_table_25) | $200,371,064 | - | - | $200,371,064 | 100% | 
| 26 | [GOAT](https://www.boxofficemojo.com/releasegroup/gr2825409285/?ref_=bo_ydw_table_26) | $195,359,397 | $103,316,898 | 52.9% | $92,042,499 | 47.1% | 
| 27 | [Insidious: Out of the Further](https://www.boxofficemojo.com/releasegroup/gr694768389/?ref_=bo_ydw_table_27) | $169,477,284 | $65,789,117 | 38.8% | $103,688,167 | 61.2% | 
| 28 | [Chiikawa the Movie: The Secret of Mermaid Island](https://www.boxofficemojo.com/releasegroup/gr3812774661/?ref_=bo_ydw_table_28) | $159,000,000 | - | - | $159,000,000 | 100% | 
| 29 | [PAW Patrol: The Dino Movie](https://www.boxofficemojo.com/releasegroup/gr2509525509/?ref_=bo_ydw_table_29) | $157,768,774 | $53,068,774 | 33.6% | $104,700,000 | 66.4% | 
| 30 | [Dhurandhar: The Revenge](https://www.boxofficemojo.com/releasegroup/gr1296651013/?ref_=bo_ydw_table_30) | $152,900,759 | $28,100,759 | 18.4% | $124,800,000 | 81.6% | 
| 31 | [Boonie Bears: The Hidden Protector](https://www.boxofficemojo.com/releasegroup/gr3259585285/?ref_=bo_ydw_table_31) | $139,300,000 | - | - | $139,300,000 | 100% | 
| 32 | [The Drama](https://www.boxofficemojo.com/releasegroup/gr913396485/?ref_=bo_ydw_table_32) | $134,670,180 | $48,102,144 | 35.7% | $86,568,036 | 64.3% | 
| 33 | [The Sheep Detectives](https://www.boxofficemojo.com/releasegroup/gr2691650309/?ref_=bo_ydw_table_33) | $133,138,836 | $66,078,506 | 49.6% | $67,060,330 | 50.4% | 
| 34 | [Mortal Kombat II](https://www.boxofficemojo.com/releasegroup/gr2607371013/?ref_=bo_ydw_table_34) | $129,570,110 | $79,770,110 | 61.6% | $49,800,000 | 38.4% | 
| 35 | [Avengers: Endgame 2026 Re-release](https://www.boxofficemojo.com/releasegroup/gr2775929605/?ref_=bo_ydw_table_35) | $126,620,026 | $36,644,026 | 28.9% | $89,976,000 | 71.1% | 
| 36 | [Supergirl](https://www.boxofficemojo.com/releasegroup/gr2573816581/?ref_=bo_ydw_table_36) | $126,466,532 | $72,366,532 | 57.2% | $54,100,000 | 42.8% | 
| 37 | [The King’s Warden](https://www.boxofficemojo.com/releasegroup/gr2353746693/?ref_=bo_ydw_table_37) | $123,083,015 | $3,574,531 | 2.9% | $119,508,484 | 97.1% | 
| 38 | [The End of Oak Street](https://www.boxofficemojo.com/releasegroup/gr1651200773/?ref_=bo_ydw_table_38) | $121,517,417 | $54,217,417 | 44.6% | $67,300,000 | 55.4% | 
| 39 | [Masters of the Universe](https://www.boxofficemojo.com/releasegroup/gr3793179141/?ref_=bo_ydw_table_39) | $113,792,300 | $64,830,245 | 57% | $48,962,055 | 43% | 
| 40 | [Practical Magic 2](https://www.boxofficemojo.com/releasegroup/gr91837189/?ref_=bo_ydw_table_40) | $100,363,603 | $65,063,603 | 64.8% | $35,300,000 | 35.2% | 
| 41 | [Heart of the Beast](https://www.boxofficemojo.com/releasegroup/gr2035176197/?ref_=bo_ydw_table_41) | $99,506,123 | $39,206,123 | 39.4% | $60,300,000 | 60.6% | 
| 42 | [Detective Conan: Fallen Angel of the Highway](https://www.boxofficemojo.com/releasegroup/gr2806993669/?ref_=bo_ydw_table_42) | $97,804,313 | - | - | $97,804,313 | 100% | 
| 43 | [Coyote vs. Acme](https://www.boxofficemojo.com/releasegroup/gr3300545029/?ref_=bo_ydw_table_43) | $97,573,795 | $59,912,137 | 61.4% | $37,661,658 | 38.6% | 
| 44 | [V](https://www.boxofficemojo.com/releasegroup/gr4266087173/?ref_=bo_ydw_table_44) | $96,232,907 | - | - | $96,232,907 | 100% | 
| 45 | [Send Help](https://www.boxofficemojo.com/releasegroup/gr3211023109/?ref_=bo_ydw_table_45) | $94,041,481 | $64,734,795 | 68.8% | $29,306,686 | 31.2% | 
| 46 | [Lee Cronin's The Mummy](https://www.boxofficemojo.com/releasegroup/gr1095258885/?ref_=bo_ydw_table_46) | $90,552,113 | $29,152,113 | 32.2% | $61,400,000 | 67.8% | 
| 47 | [V](https://www.boxofficemojo.com/releasegroup/gr3426767621/?ref_=bo_ydw_table_47) | $89,600,000 | - | - | $89,600,000 | 100% | 
| 48 | [Reminders of Him](https://www.boxofficemojo.com/releasegroup/gr2908902149/?ref_=bo_ydw_table_48) | $89,274,157 | $48,559,430 | 54.4% | $40,714,727 | 45.6% | 
| 49 | [Vanishing Point](https://www.boxofficemojo.com/releasegroup/gr1850626821/?ref_=bo_ydw_table_49) | $81,200,457 | - | - | $81,200,457 | 100% | 
| 50 | [Crime 101](https://www.boxofficemojo.com/releasegroup/gr276058885/?ref_=bo_ydw_table_50) | $72,900,246 | $36,588,336 | 50.2% | $36,311,910 | 49.8% | 
| 51 | [Evil Dead Burn](https://www.boxofficemojo.com/releasegroup/gr3531100933/?ref_=bo_ydw_table_51) | $72,377,053 | $32,237,058 | 44.5% | $40,139,995 | 55.5% | 
| 52 | [Verity](https://www.boxofficemojo.com/releasegroup/gr343626501/?ref_=bo_ydw_table_52) | $64,256,570 | $34,433,247 | 53.6% | $29,823,323 | 46.4% | 
| 53 | [The Invite](https://www.boxofficemojo.com/releasegroup/gr709513989/?ref_=bo_ydw_table_53) | $59,952,135 | $26,099,131 | 43.5% | $33,853,004 | 56.5% | 
| 54 | [28 Years Later: The Bone Temple](https://www.boxofficemojo.com/releasegroup/gr3982906117/?ref_=bo_ydw_table_54) | $58,527,279 | $25,147,583 | 43% | $33,379,696 | 57% | 
| 55 | [Shelter](https://www.boxofficemojo.com/releasegroup/gr4218508037/?ref_=bo_ydw_table_55) | $56,171,324 | $12,805,541 | 22.8% | $43,365,783 | 77.2% | 
| 56 | [Mercy](https://www.boxofficemojo.com/releasegroup/gr1617253125/?ref_=bo_ydw_table_56) | $54,709,856 | $24,390,303 | 44.6% | $30,319,553 | 55.4% | 
| 57 | [Colony](https://www.boxofficemojo.com/releasegroup/gr3645854469/?ref_=bo_ydw_table_57) | $53,128,238 | $1,456,674 | 2.7% | $51,671,564 | 97.3% | 
| 58 | [Marsupilami](https://www.boxofficemojo.com/releasegroup/gr323703557/?ref_=bo_ydw_table_58) | $52,840,695 | - | - | $52,840,695 | 100% | 
| 59 | [Iron Lung](https://www.boxofficemojo.com/releasegroup/gr2034979589/?ref_=bo_ydw_table_59) | $50,036,487 | $40,851,941 | 81.6% | $9,184,546 | 18.4% | 
| 60 | [Return to Silent Hill](https://www.boxofficemojo.com/releasegroup/gr91771653/?ref_=bo_ydw_table_60) | $47,533,175 | $5,544,971 | 11.7% | $41,988,204 | 88.3% | 
| 61 | [Young Washington](https://www.boxofficemojo.com/releasegroup/gr3041481477/?ref_=bo_ydw_table_61) | $47,149,820 | $47,075,084 | 99.8% | $74,736 | 0.2% | 
| 62 | [Cold War 1994](https://www.boxofficemojo.com/releasegroup/gr2320388869/?ref_=bo_ydw_table_62) | $46,907,049 | $361,156 | 0.8% | $46,545,893 | 99.2% | 
| 63 | [Hope](https://www.boxofficemojo.com/releasegroup/gr3007533829/?ref_=bo_ydw_table_63) | $46,803,697 | $8,137,518 | 17.4% | $38,666,179 | 82.6% | 
| 64 | [Greenland 2: Migration](https://www.boxofficemojo.com/releasegroup/gr444420869/?ref_=bo_ydw_table_64) | $45,191,228 | $17,770,308 | 39.3% | $27,420,920 | 60.7% | 
| 65 | [Ready or Not 2: Here I Come](https://www.boxofficemojo.com/releasegroup/gr2491372293/?ref_=bo_ydw_table_65) | $43,096,054 | $22,986,054 | 53.3% | $20,110,000 | 46.7% | 
| 66 | [Kingdom 5](https://www.boxofficemojo.com/releasegroup/gr625169157/?ref_=bo_ydw_table_66) | $42,000,782 | - | - | $42,000,782 | 100% | 
| 67 | [Traces of Justice](https://www.boxofficemojo.com/releasegroup/gr1850168069/?ref_=bo_ydw_table_67) | $41,800,000 | - | - | $41,800,000 | 100% | 
| 68 | [The Furious](https://www.boxofficemojo.com/releasegroup/gr3326628613/?ref_=bo_ydw_table_68) | $41,764,290 | $6,502,610 | 15.6% | $35,261,680 | 84.4% | 
| 69 | [Crossing](https://www.boxofficemojo.com/releasegroup/gr3897316101/?ref_=bo_ydw_table_69) | $41,517,740 | $37,181 | \<0.1% | $41,480,559 | 99.9% | 
| 70 | [The Dog Stars](https://www.boxofficemojo.com/releasegroup/gr3950990085/?ref_=bo_ydw_table_70) | $39,321,776 | $14,961,776 | 38% | $24,360,000 | 62% | 
| 71 | [Primate](https://www.boxofficemojo.com/releasegroup/gr1165775621/?ref_=bo_ydw_table_71) | $39,074,453 | $25,635,665 | 65.6% | $13,438,788 | 34.4% | 
| 72 | [Forgotten Island](https://www.boxofficemojo.com/releasegroup/gr310006533/?ref_=bo_ydw_table_72) | $39,011,870 | $24,115,870 | 61.8% | $14,896,000 | 38.2% | 
| 73 | [The Amazing Digital Circus: The Last Act](https://www.boxofficemojo.com/releasegroup/gr1280135941/?ref_=bo_ydw_table_73) | $36,405,571 | $23,421,793 | 64.3% | $12,983,778 | 35.7% | 
| 74 | [The Magic Faraway Tree](https://www.boxofficemojo.com/releasegroup/gr4266152709/?ref_=bo_ydw_table_74) | $34,729,236 | $2,391,837 | 6.9% | $32,337,399 | 93.1% | 
| 75 | [Primetime](https://www.boxofficemojo.com/releasegroup/gr1044599557/?ref_=bo_ydw_table_75) | $34,189,539 | $34,085,832 | 99.7% | $103,707 | 0.3% | 
| 76 | [Torrente for President](https://www.boxofficemojo.com/releasegroup/gr1464423173/?ref_=bo_ydw_table_76) | $33,080,103 | - | - | $33,080,103 | 100% | 
| 77 | [Passenger](https://www.boxofficemojo.com/releasegroup/gr2138329861/?ref_=bo_ydw_table_77) | $31,620,508 | $18,041,122 | 57.1% | $13,579,386 | 42.9% | 
| 78 | [A Werewolf Boy](https://www.boxofficemojo.com/releasegroup/gr910906117/?ref_=bo_ydw_table_78) | $30,940,545 | - | - | $30,940,545 | 100% | 
| 79 | [Mutiny](https://www.boxofficemojo.com/releasegroup/gr2523157253/?ref_=bo_ydw_table_79) | $30,486,896 | $15,835,475 | 51.9% | $14,651,421 | 48.1% | 
| 80 | [Harry Potter and the Deathly Hallows: Part 1 2026 Re-release](https://www.boxofficemojo.com/releasegroup/gr1749111557/?ref_=bo_ydw_table_80) | $30,354,696 | - | - | $30,354,696 | 100% | 
| 81 | [Hanuman Ansh](https://www.boxofficemojo.com/releasegroup/gr2202030853/?ref_=bo_ydw_table_81) | $30,232,308 | $2,332,308 | 7.7% | $27,900,000 | 92.3% | 
| 82 | [Pressure](https://www.boxofficemojo.com/releasegroup/gr1766544133/?ref_=bo_ydw_table_82) | $29,044,521 | $15,569,855 | 53.6% | $13,474,666 | 46.4% | 
| 83 | [In the Grey](https://www.boxofficemojo.com/releasegroup/gr913003269/?ref_=bo_ydw_table_83) | $28,431,706 | $5,723,890 | 20.1% | $22,707,816 | 79.9% | 
| 84 | [Drishyam: The Conclusion](https://www.boxofficemojo.com/releasegroup/gr2017940229/?ref_=bo_ydw_table_84) | $28,215,000 | $2,715,000 | 9.6% | $25,500,000 | 90.4% | 
| 85 | [It's OK](https://www.boxofficemojo.com/releasegroup/gr1397707525/?ref_=bo_ydw_table_85) | $28,080,000 | - | - | $28,080,000 | 100% | 
| 86 | [Jackass: Best and Last](https://www.boxofficemojo.com/releasegroup/gr3444265733/?ref_=bo_ydw_table_86) | $27,830,990 | $18,273,185 | 65.7% | $9,557,805 | 34.3% | 
| 87 | [Until We Meet Again](https://www.boxofficemojo.com/releasegroup/gr3444200197/?ref_=bo_ydw_table_87) | $27,679,394 | - | - | $27,679,394 | 100% | 
| 88 | [The Tale of Tsar Saltan](https://www.boxofficemojo.com/releasegroup/gr1883919109/?ref_=bo_ydw_table_88) | $27,565,988 | - | - | $27,565,988 | 100% | 
| 89 | [Doraemon: Nobita and the New Castle of the Undersea Devil](https://www.boxofficemojo.com/releasegroup/gr2655605509/?ref_=bo_ydw_table_89) | $27,322,038 | - | - | $27,322,038 | 100% | 
| 90 | [Buddy](https://www.boxofficemojo.com/releasegroup/gr1129272069/?ref_=bo_ydw_table_90) | $26,869,263 | $26,869,263 | 100% | - | - | 
| 91 | [Billie Eilish: Hit Me Hard and Soft](https://www.boxofficemojo.com/releasegroup/gr276189957/?ref_=bo_ydw_table_91) | $26,721,893 | $9,901,485 | 37.1% | $16,820,408 | 62.9% | 
| 92 | [Solo Mio](https://www.boxofficemojo.com/releasegroup/gr142365445/?ref_=bo_ydw_table_92) | $26,433,346 | $25,736,278 | 97.4% | $697,068 | 2.6% | 
| 93 | [One Night Only](https://www.boxofficemojo.com/releasegroup/gr3296482053/?ref_=bo_ydw_table_93) | $25,121,894 | $11,506,280 | 45.8% | $13,615,614 | 54.2% | 
| 94 | [Hokum](https://www.boxofficemojo.com/releasegroup/gr2622051077/?ref_=bo_ydw_table_94) | $25,023,390 | $17,000,014 | 67.9% | $8,023,376 | 32.1% | 
| 95 | [Mr. & Mrs. Pardon](https://www.boxofficemojo.com/releasegroup/gr1246188293/?ref_=bo_ydw_table_95) | $24,500,000 | - | - | $24,500,000 | 100% | 
| 96 | [A Part of Me](https://www.boxofficemojo.com/releasegroup/gr54743813/?ref_=bo_ydw_table_96) | $24,368,934 | - | - | $24,368,934 | 100% | 
| 97 | [The Bride!](https://www.boxofficemojo.com/releasegroup/gr1570001413/?ref_=bo_ydw_table_97) | $23,944,025 | $12,744,025 | 53.2% | $11,200,000 | 46.8% | 
| 98 | [Stray Kids: The Dominate Experience](https://www.boxofficemojo.com/releasegroup/gr21713669/?ref_=bo_ydw_table_98) | $23,465,818 | $7,232,818 | 30.8% | $16,233,000 | 69.2% | 
| 99 | [Epic: Elvis Presley in Concert](https://www.boxofficemojo.com/releasegroup/gr2507494149/?ref_=bo_ydw_table_99) | $23,463,883 | $13,571,883 | 57.8% | $9,892,000 | 42.2% | 
| 100 | [You, Me & Tuscany](https://www.boxofficemojo.com/releasegroup/gr2323206917/?ref_=bo_ydw_table_100) | $22,456,969 | $18,723,685 | 83.4% | $3,733,284 | 16.6% | 
| 101 | [Salmokji: Whispering Water](https://www.boxofficemojo.com/releasegroup/gr1867010821/?ref_=bo_ydw_table_101) | $22,032,184 | - | - | $22,032,184 | 100% | 
| 102 | [Extrawurst](https://www.boxofficemojo.com/releasegroup/gr3612037893/?ref_=bo_ydw_table_102) | $21,927,663 | - | - | $21,927,663 | 100% | 
| 103 | [How to Make a Killing](https://www.boxofficemojo.com/releasegroup/gr1081496325/?ref_=bo_ydw_table_103) | $21,655,441 | $7,851,600 | 36.3% | $13,803,841 | 63.7% | 
| 104 | [Undertone](https://www.boxofficemojo.com/releasegroup/gr846287621/?ref_=bo_ydw_table_104) | $21,555,603 | $20,001,151 | 92.8% | $1,554,452 | 7.2% | 
| 105 | [Hadestown: The Musical](https://www.boxofficemojo.com/releasegroup/gr1396921093/?ref_=bo_ydw_table_105) | $20,728,068 | $20,728,068 | 100% | - | - | 
| 106 | [Digger](https://www.boxofficemojo.com/releasegroup/gr2338738949/?ref_=bo_ydw_table_106) | $20,568,027 | $8,768,027 | 42.6% | $11,800,000 | 57.4% | 
| 107 | [De Gaulle: Résistance](https://www.boxofficemojo.com/releasegroup/gr1531794181/?ref_=bo_ydw_table_107) | $20,486,328 | - | - | $20,486,328 | 100% | 
| 108 | [The Breadwinner](https://www.boxofficemojo.com/releasegroup/gr3497677573/?ref_=bo_ydw_table_108) | $20,247,080 | $20,247,080 | 100% | - | - | 
| 109 | [Cosmic Princess Kaguya!](https://www.boxofficemojo.com/releasegroup/gr3460911877/?ref_=bo_ydw_table_109) | $20,227,966 | - | - | $20,227,966 | 100% | 
| 110 | [Keep Real](https://www.boxofficemojo.com/releasegroup/gr4098052869/?ref_=bo_ydw_table_110) | $19,500,000 | - | - | $19,500,000 | 100% | 
| 111 | [They Will Kill You](https://www.boxofficemojo.com/releasegroup/gr2540786437/?ref_=bo_ydw_table_111) | $19,382,152 | $10,882,152 | 56.1% | $8,500,000 | 43.9% | 
| 112 | [Cars 2026 Re-release](https://www.boxofficemojo.com/releasegroup/gr3309392645/?ref_=bo_ydw_table_112) | $19,121,451 | $11,076,228 | 57.9% | $8,045,223 | 42.1% | 
| 113 | [Tony](https://www.boxofficemojo.com/releasegroup/gr3662566149/?ref_=bo_ydw_table_113) | $19,097,303 | $15,474,121 | 81% | $3,623,182 | 19% | 
| 114 | [I Can Only Imagine 2](https://www.boxofficemojo.com/releasegroup/gr1367036677/?ref_=bo_ydw_table_114) | $18,741,530 | $18,588,615 | 99.2% | $152,915 | 0.8% | 
| 115 | [Mobile Suit Gundam Hathaway: The Sorcery of Nymph Circe](https://www.boxofficemojo.com/releasegroup/gr122376965/?ref_=bo_ydw_table_115) | $18,420,625 | $1,165,491 | 6.3% | $17,255,134 | 93.7% | 
| 116 | [Steckerlfischfiasko](https://www.boxofficemojo.com/releasegroup/gr3846263557/?ref_=bo_ydw_table_116) | $18,218,370 | - | - | $18,218,370 | 100% | 
| 117 | [Runner](https://www.boxofficemojo.com/releasegroup/gr2638435077/?ref_=bo_ydw_table_117) | $17,720,202 | $14,351,184 | 81% | $3,369,018 | 19% | 
| 118 | [Crayon Shin-chan the Movie: Spooky! My Yokai Vocation](https://www.boxofficemojo.com/releasegroup/gr1631736581/?ref_=bo_ydw_table_118) | $17,511,097 | - | - | $17,511,097 | 100% | 
| 119 | [Bunny!!](https://www.boxofficemojo.com/releasegroup/gr172643077/?ref_=bo_ydw_table_119) | $17,475,856 | $669,093 | 3.8% | $16,806,763 | 96.2% | 
| 120 | [Guru](https://www.boxofficemojo.com/releasegroup/gr676025093/?ref_=bo_ydw_table_120) | $16,836,426 | - | - | $16,836,426 | 100% | 
| 121 | [Bayside Shakedown N.E.W.](https://www.boxofficemojo.com/releasegroup/gr3477558021/?ref_=bo_ydw_table_121) | $16,828,565 | - | - | $16,828,565 | 100% | 
| 122 | [By Any Means](https://www.boxofficemojo.com/releasegroup/gr3863958277/?ref_=bo_ydw_table_122) | $16,728,197 | $15,763,404 | 94.2% | $964,793 | 5.8% | 
| 123 | [Melania](https://www.boxofficemojo.com/releasegroup/gr2051756805/?ref_=bo_ydw_table_123) | $16,661,439 | $16,357,453 | 98.2% | $303,986 | 1.8% | 
| 124 | [Sakamoto Days](https://www.boxofficemojo.com/releasegroup/gr2521715461/?ref_=bo_ydw_table_124) | $16,595,031 | - | - | $16,595,031 | 100% | 
| 125 | [I Know Who You Are](https://www.boxofficemojo.com/releasegroup/gr2941014789/?ref_=bo_ydw_table_125) | $16,364,188 | - | - | $16,364,188 | 100% | 
| 126 | [La La Land 2026 Re-release](https://www.boxofficemojo.com/releasegroup/gr2369868549/?ref_=bo_ydw_table_126) | $16,329,051 | - | - | $16,329,051 | 100% | 
| 127 | [Just an Illusion](https://www.boxofficemojo.com/releasegroup/gr1280267013/?ref_=bo_ydw_table_127) | $15,774,023 | - | - | $15,774,023 | 100% | 
| 128 | [The Caged Butterfly](https://www.boxofficemojo.com/releasegroup/gr1380930309/?ref_=bo_ydw_table_128) | $15,210,000 | - | - | $15,210,000 | 100% | 
| 129 | [Fall 2: Deadpoint](https://www.boxofficemojo.com/releasegroup/gr2337166085/?ref_=bo_ydw_table_129) | $15,202,310 | $5,044,421 | 33.2% | $10,157,889 | 66.8% | 
| 130 | [Top Gun/Top Gun: Maverick 2026 Re-release (Top Gun 40th Anniversary)](https://www.boxofficemojo.com/releasegroup/gr3578352389/?ref_=bo_ydw_table_130) | $15,128,326 | $6,528,326 | 43.2% | $8,600,000 | 56.8% | 
| 131 | [Hypnosismic -Division Rap Battle- Rule the Stage -Battle of Pride 2023](https://www.boxofficemojo.com/releasegroup/gr309547781/?ref_=bo_ydw_table_131) | $15,069,212 | $58,276 | 0.4% | $15,010,936 | 99.6% | 
| 132 | [To Your Island](https://www.boxofficemojo.com/releasegroup/gr2067944197/?ref_=bo_ydw_table_132) | $15,000,000 | - | - | $15,000,000 | 100% | 
| 133 | [The Three Investigators: Isle of Death](https://www.boxofficemojo.com/releasegroup/gr2286637829/?ref_=bo_ydw_table_133) | $14,911,324 | - | - | $14,911,324 | 100% | 
| 134 | [Puella Magi Madoka Magica the Movie -Walpurgisnacht Rising-](https://www.boxofficemojo.com/releasegroup/gr3862975237/?ref_=bo_ydw_table_134) | $14,838,316 | - | - | $14,838,316 | 100% | 
| 135 | [Tuner](https://www.boxofficemojo.com/releasegroup/gr1014256389/?ref_=bo_ydw_table_135) | $14,613,540 | $4,511,186 | 30.9% | $10,102,354 | 69.1% | 
| 136 | [Kyojo: Requiem](https://www.boxofficemojo.com/releasegroup/gr3377025797/?ref_=bo_ydw_table_136) | $14,585,945 | - | - | $14,585,945 | 100% | 
| 137 | [Night King](https://www.boxofficemojo.com/releasegroup/gr222974725/?ref_=bo_ydw_table_137) | $14,566,269 | - | - | $14,566,269 | 100% | 
| 138 | [Kholop 3](https://www.boxofficemojo.com/releasegroup/gr3427554053/?ref_=bo_ydw_table_138) | $14,483,813 | - | - | $14,483,813 | 100% | 
| 139 | [De Gaulle: Liberté](https://www.boxofficemojo.com/releasegroup/gr1129075461/?ref_=bo_ydw_table_139) | $14,320,015 | - | - | $14,320,015 | 100% | 
| 140 | [Ach, diese Lücke, diese entsetzliche Lücke](https://www.boxofficemojo.com/releasegroup/gr1430999813/?ref_=bo_ydw_table_140) | $14,134,508 | - | - | $14,134,508 | 100% | 
| 141 | [Svadba](https://www.boxofficemojo.com/releasegroup/gr2303415045/?ref_=bo_ydw_table_141) | $13,817,137 | - | - | $13,817,137 | 100% | 
| 142 | [Spa Weekend](https://www.boxofficemojo.com/releasegroup/gr846484229/?ref_=bo_ydw_table_142) | $13,768,989 | $7,016,616 | 51% | $6,752,373 | 49% | 
| 143 | [Humint](https://www.boxofficemojo.com/releasegroup/gr508252933/?ref_=bo_ydw_table_143) | $13,207,600 | - | - | $13,207,600 | 100% | 
| 144 | [Tad and the Magic Lamp](https://www.boxofficemojo.com/releasegroup/gr2189120261/?ref_=bo_ydw_table_144) | $13,078,013 | - | - | $13,078,013 | 100% | 
| 145 | [Awarapan 2](https://www.boxofficemojo.com/releasegroup/gr1682068229/?ref_=bo_ydw_table_145) | $12,800,000 | - | - | $12,800,000 | 100% | 
| 146 | [Don't Look Back in Anger](https://www.boxofficemojo.com/releasegroup/gr525095685/?ref_=bo_ydw_table_146) | $12,485,044 | $1,961,044 | 15.7% | $10,524,000 | 84.3% | 
| 147 | [Little Brownie Kuzya 2](https://www.boxofficemojo.com/releasegroup/gr3477623557/?ref_=bo_ydw_table_147) | $12,451,384 | - | - | $12,451,384 | 100% | 
| 148 | [Vishwanath and Sons](https://www.boxofficemojo.com/releasegroup/gr121787141/?ref_=bo_ydw_table_148) | $12,334,000 | $1,634,000 | 13.2% | $10,700,000 | 86.8% | 
| 149 | [Your Heart Will Be Broken](https://www.boxofficemojo.com/releasegroup/gr1179144965/?ref_=bo_ydw_table_149) | $12,318,524 | - | - | $12,318,524 | 100% | 
| 150 | [Posledniy bogatyr. Kolobok](https://www.boxofficemojo.com/releasegroup/gr1296192261/?ref_=bo_ydw_table_150) | $12,261,848 | - | - | $12,261,848 | 100% | 
| 151 | [The Three Kingdoms Part 1: The Battle for Luoyang](https://www.boxofficemojo.com/releasegroup/gr3460518661/?ref_=bo_ydw_table_151) | $12,200,000 | - | - | $12,200,000 | 100% | 
| 152 | [Woodwalkers 2](https://www.boxofficemojo.com/releasegroup/gr1414222597/?ref_=bo_ydw_table_152) | $11,875,930 | - | - | $11,875,930 | 100% | 
| 153 | [The Eyes](https://www.boxofficemojo.com/releasegroup/gr239883013/?ref_=bo_ydw_table_153) | $11,249,071 | - | - | $11,249,071 | 100% | 
| 154 | [The Strangers: Chapter 3](https://www.boxofficemojo.com/releasegroup/gr125195013/?ref_=bo_ydw_table_154) | $11,158,087 | $8,560,986 | 76.7% | $2,597,101 | 23.3% | 
| 155 | [The Decisive Moment](https://www.boxofficemojo.com/releasegroup/gr3779220229/?ref_=bo_ydw_table_155) | $11,008,292 | - | - | $11,008,292 | 100% | 
| 156 | [Now I Met Her](https://www.boxofficemojo.com/releasegroup/gr3981398789/?ref_=bo_ydw_table_156) | $10,950,000 | - | - | $10,950,000 | 100% | 
| 157 | [Be Yourself](https://www.boxofficemojo.com/releasegroup/gr2706199301/?ref_=bo_ydw_table_157) | $10,600,000 | - | - | $10,600,000 | 100% | 
| 158 | [Children of the Resistance](https://www.boxofficemojo.com/releasegroup/gr1732924165/?ref_=bo_ydw_table_158) | $10,518,093 | - | - | $10,518,093 | 100% | 
| 159 | [Hakucho to Komori](https://www.boxofficemojo.com/releasegroup/gr1631605509/?ref_=bo_ydw_table_159) | $10,299,391 | - | - | $10,299,391 | 100% | 
| 160 | [Being Towards Death](https://www.boxofficemojo.com/releasegroup/gr1817072389/?ref_=bo_ydw_table_160) | $10,203,144 | - | - | $10,203,144 | 100% | 
| 161 | [Santiago: The Camino Therapy](https://www.boxofficemojo.com/releasegroup/gr21517061/?ref_=bo_ydw_table_161) | $10,003,391 | - | - | $10,003,391 | 100% | 
| 162 | [Good Luck, Have Fun, Don't Die](https://www.boxofficemojo.com/releasegroup/gr997413637/?ref_=bo_ydw_table_162) | $9,952,131 | $8,431,443 | 84.7% | $1,520,688 | 15.3% | 
| 163 | [The Death of Robin Hood](https://www.boxofficemojo.com/releasegroup/gr1565217541/?ref_=bo_ydw_table_163) | $9,896,460 | $5,355,164 | 54.1% | $4,541,296 | 45.9% | 
| 164 | [Gintama: Yoshiwara in Flames](https://www.boxofficemojo.com/releasegroup/gr1397379845/?ref_=bo_ydw_table_164) | $9,888,965 | - | - | $9,888,965 | 100% | 
| 165 | [I Love Boosters](https://www.boxofficemojo.com/releasegroup/gr3074970373/?ref_=bo_ydw_table_165) | $9,743,067 | $9,743,067 | 100% | - | - | 
| 166 | [The Money Maker](https://www.boxofficemojo.com/releasegroup/gr2840285957/?ref_=bo_ydw_table_166) | $9,630,941 | - | - | $9,630,941 | 100% | 
| 167 | [Wild Sing](https://www.boxofficemojo.com/releasegroup/gr2622313221/?ref_=bo_ydw_table_167) | $9,025,511 | - | - | $9,025,511 | 100% | 
| 168 | [BTS World Tour 'Arirang' in Busan: Live Viewing](https://www.boxofficemojo.com/releasegroup/gr139350789/?ref_=bo_ydw_table_168) | $8,981,421 | $3,800,000 | 42.3% | $5,181,421 | 57.7% | 
| 169 | [Golden Kamuy: Assault on Abashiri Prison](https://www.boxofficemojo.com/releasegroup/gr172577541/?ref_=bo_ydw_table_169) | $8,971,729 | - | - | $8,971,729 | 100% | 
| 170 | [Leviticus](https://www.boxofficemojo.com/releasegroup/gr3494466309/?ref_=bo_ydw_table_170) | $8,905,776 | $7,215,533 | 81% | $1,690,243 | 19% | 
| 171 | [Shaun the Sheep: The Beast of Mossy Bottom](https://www.boxofficemojo.com/releasegroup/gr927552261/?ref_=bo_ydw_table_171) | $8,824,997 | $3,054,790 | 34.6% | $5,770,207 | 65.4% | 
| 172 | [Teenage Sex and Death at Camp Miasma](https://www.boxofficemojo.com/releasegroup/gr474632965/?ref_=bo_ydw_table_172) | $8,675,399 | $6,833,898 | 78.8% | $1,841,501 | 21.2% | 
| 173 | [The Wizard of the Kremlin](https://www.boxofficemojo.com/releasegroup/gr2219528965/?ref_=bo_ydw_table_173) | $8,581,956 | $652,300 | 7.6% | $7,929,656 | 92.4% | 
| 174 | [The Things Left Unspoken](https://www.boxofficemojo.com/releasegroup/gr1581994757/?ref_=bo_ydw_table_174) | $8,188,844 | - | - | $8,188,844 | 100% | 
| 175 | [The Uprising](https://www.boxofficemojo.com/releasegroup/gr1833915141/?ref_=bo_ydw_table_175) | $8,151,392 | $6,513,180 | 79.9% | $1,638,212 | 20.1% | 
| 176 | [A Great Awakening](https://www.boxofficemojo.com/releasegroup/gr659182341/?ref_=bo_ydw_table_176) | $8,149,028 | $8,149,028 | 100% | - | - | 
| 177 | [Deep Water](https://www.boxofficemojo.com/releasegroup/gr307188485/?ref_=bo_ydw_table_177) | $8,130,398 | $4,346,226 | 53.5% | $3,784,172 | 46.5% | 
| 178 | [Drishyam 3](https://www.boxofficemojo.com/releasegroup/gr3830272773/?ref_=bo_ydw_table_178) | $8,102,169 | $1,650,583 | 20.4% | $6,451,586 | 79.6% | 
| 179 | [Checker Tobi 3: Die heimliche Herrscherin der Erde](https://www.boxofficemojo.com/releasegroup/gr913593093/?ref_=bo_ydw_table_179) | $8,017,475 | - | - | $8,017,475 | 100% | 
| 180 | [Super Troopers 3](https://www.boxofficemojo.com/releasegroup/gr2759152389/?ref_=bo_ydw_table_180) | $7,779,852 | $7,435,972 | 95.6% | $343,880 | 4.4% | 
| 181 | [LOL 2.0](https://www.boxofficemojo.com/releasegroup/gr1682592517/?ref_=bo_ydw_table_181) | $7,753,402 | - | - | $7,753,402 | 100% | 
| 182 | [Game Start](https://www.boxofficemojo.com/releasegroup/gr3796849413/?ref_=bo_ydw_table_182) | $7,750,000 | - | - | $7,750,000 | 100% | 
| 183 | [Dear Parents](https://www.boxofficemojo.com/releasegroup/gr3511243525/?ref_=bo_ydw_table_183) | $7,718,850 | - | - | $7,718,850 | 100% | 
| 184 | [Daniel and the Fiery Furnace](https://www.boxofficemojo.com/releasegroup/gr2403554053/?ref_=bo_ydw_table_184) | $7,649,486 | $7,649,486 | 100% | - | - | 
| 185 | [Ice Cream Man](https://www.boxofficemojo.com/releasegroup/gr2571653893/?ref_=bo_ydw_table_185) | $7,607,037 | $3,174,200 | 41.7% | $4,432,837 | 58.3% | 
| 186 | [The Lord of the Rings: The Fellowship of the Ring 2026  Re-release](https://www.boxofficemojo.com/releasegroup/gr1970295557/?ref_=bo_ydw_table_186) | $7,481,290 | $5,940,331 | 79.4% | $1,486,329 | 19.9% | 
| 187 | [Good Vibes Only](https://www.boxofficemojo.com/releasegroup/gr1280201477/?ref_=bo_ydw_table_187) | $7,476,473 | - | - | $7,476,473 | 100% | 
| 188 | [The Rays and Shadows](https://www.boxofficemojo.com/releasegroup/gr3494400773/?ref_=bo_ydw_table_188) | $7,440,895 | - | - | $7,440,895 | 100% | 
| 189 | [Bethlehem Kudumba Unit](https://www.boxofficemojo.com/releasegroup/gr3594605317/?ref_=bo_ydw_table_189) | $7,401,491 | $813,569 | 11% | $6,587,922 | 89% | 
| 190 | [That Time I Got Reincarnated as a Slime the Movie: Tears of the Azure Sea](https://www.boxofficemojo.com/releasegroup/gr508187397/?ref_=bo_ydw_table_190) | $7,400,334 | $1,423,753 | 19.2% | $5,976,581 | 80.8% | 
| 191 | [The Boy Who Counted Cars](https://www.boxofficemojo.com/releasegroup/gr2890683141/?ref_=bo_ydw_table_191) | $7,400,000 | - | - | $7,400,000 | 100% | 
| 192 | [Horst Schlämmer Is Looking for Happiness](https://www.boxofficemojo.com/releasegroup/gr1783124741/?ref_=bo_ydw_table_192) | $7,376,329 | - | - | $7,376,329 | 100% | 
| 193 | [The Samurai and the Prisoner](https://www.boxofficemojo.com/releasegroup/gr290345733/?ref_=bo_ydw_table_193) | $7,138,747 | $838,112 | 11.7% | $6,300,635 | 88.3% | 
| 194 | [Katseye: Wild Hearts](https://www.boxofficemojo.com/releasegroup/gr2873316101/?ref_=bo_ydw_table_194) | $6,956,009 | $4,629,744 | 66.6% | $2,326,265 | 33.4% | 
| 195 | [BTS World Tour 'Arirang' in Goyang: Live Viewing](https://www.boxofficemojo.com/releasegroup/gr2135577349/?ref_=bo_ydw_table_195) | $6,936,152 | $2,440,000 | 35.2% | $4,496,152 | 64.8% | 
| 196 | [Yetimhane: Sahipsiz Cinler](https://www.boxofficemojo.com/releasegroup/gr3410907909/?ref_=bo_ydw_table_196) | $6,909,959 | - | - | $6,909,959 | 100% | 
| 197 | [Star Detective Precure!: The Mysterious Garden and the Pair's Secret](https://www.boxofficemojo.com/releasegroup/gr3460780805/?ref_=bo_ydw_table_197) | $6,895,418 | - | - | $6,895,418 | 100% | 
| 198 | [Animal Farm](https://www.boxofficemojo.com/releasegroup/gr1450529541/?ref_=bo_ydw_table_198) | $6,678,620 | $5,588,982 | 83.7% | $1,089,638 | 16.3% | 
| 199 | [Kiki's Delivery Service 2026 Re-release](https://www.boxofficemojo.com/releasegroup/gr608850693/?ref_=bo_ydw_table_199) | $6,633,379 | $2,697,212 | 40.7% | $3,936,167 | 59.3% | 
| 200 | [Kommersant](https://www.boxofficemojo.com/releasegroup/gr1028608773/?ref_=bo_ydw_table_200) | $6,573,805 | - | - | $6,573,805 | 100% |
[10/07/26 19:51:51] INFO     Processing request of type           server.py:733
                             CallToolRequest                                   
[10/07/26 19:51:52] INFO     HTTP Request: GET                  _client.py:1740
                             https://www.boxofficemojo.com/year                
                             /world/ "HTTP/1.1 200 OK"                         
[10/07/26 19:51:53] INFO     Processing request of type           server.py:733
                             ListToolsRequest                                  
MiniAgent Acting >mcp__web_read的执行结果：## 2026 Worldwide Box Office - Box Office Mojo

> URL: https://www.boxofficemojo.com/year/world/ | Method: trafilatura | Sections: 0 | Quality: 58% (density: 92%, structure: 0%, clean: 100%, complete: 0%)

| [Rank](https://www.boxofficemojo.com?sort=rank&ref_=bo_ydw__resort#table) | Release Group | [Worldwide](https://www.boxofficemojo.com?sort=worldwideGrossToDate&ref_=bo_ydw__resort#table) | [Domestic](https://www.boxofficemojo.com?sort=domesticGrossToDate&ref_=bo_ydw__resort#table) | [%](https://www.boxofficemojo.com?sort=domesticGrossToDatePercent&ref_=bo_ydw__resort#table) | [Foreign](https://www.boxofficemojo.com?sort=foreignGrossToDate&ref_=bo_ydw__resort#table) | [%](https://www.boxofficemojo.com?sort=foreignGrossToDatePercent&ref_=bo_ydw__resort#table) | 
这是第 3 次循环...
MiniAgent observation ...
MiniAgent Reasoning > 我已获得前5名票房。现在需要每部的上映时间（release date）。需要逐个读取 releasegroup 页面，或用 IMDb 查询。Box Office Mojo releasegroup 页面有 "Release Date"。

前5名：
1. Spider-Man: Brand New Day — gr3277935365
2. The Odyssey — gr359027461
3. Toy Story 5 — gr594236165
4. Michael — gr3549843973
5. The Super Mario Galaxy Movie — gr513036805

让我并行读取这5个页面。但可能每个页面内容较大。用 mode=preview 或 full 但 max_length 小一些。Box Office Mojo releasegroup 页面通常包含 "Release Date" 和类型、制作预算等。

先并行读5个。

[10/07/26 19:52:42] INFO     Processing request of type           server.py:733
                             CallToolRequest                                   
[10/07/26 19:52:44] INFO     HTTP Request: GET                  _client.py:1740
                             https://www.boxofficemojo.com/rele                
                             asegroup/gr3277935365/ "HTTP/1.1                  
                             200 OK"                                           
[10/07/26 19:52:45] INFO     Processing request of type           server.py:733
                             ListToolsRequest                                  
MiniAgent Acting >mcp__web_read的执行结果：## Spider-Man: Brand New Day - Box Office Mojo

> URL: https://www.boxofficemojo.com/releasegroup/gr3277935365/ | Method: trafilatura | Quality: 68% (density: 92%, structure: 1%, clean: 100%, complete: 70%) | cache:MISS

# Spider-Man: Brand New Day

A forgotten Peter Parker lives alone as a full-time Spider-Man until mounting pressure triggers a dangerous change and a powerful new enemy emerges.

| Domestic |  |  |  | 
|---|---|---|---|
| Market | Release Date | Opening | Gross | 
| [Domestic](https://www.boxofficemojo.com/release/rl2299756545/?ref_=bo_gr_rls) | Jul 31, 2026 | $360,091,572 | $954,734,627 | 

> [Truncated: 5654 -> 393 chars, 6% retained]
[10/07/26 19:52:48] INFO     Processing request of type           server.py:733
                             CallToolRequest                                   
[10/07/26 19:52:49] INFO     HTTP Request: GET                  _client.py:1740
                             https://www.boxofficemojo.com/rele                
                             asegroup/gr359027461/ "HTTP/1.1                   
                             200 OK"                                           
[10/07/26 19:52:50] INFO     Processing request of type           server.py:733
                             ListToolsRequest                                  
MiniAgent Acting >mcp__web_read的执行结果：## The Odyssey - Box Office Mojo

> URL: https://www.boxofficemojo.com/releasegroup/gr359027461/ | Method: trafilatura | Quality: 68% (density: 93%, structure: 1%, clean: 100%, complete: 70%) | cache:MISS

# The Odyssey

After the Trojan War, Odysseus faces a dangerous voyage back to Ithaca, meeting creatures like the Cyclops Polyphemus, Sirens, and Calypso along the way.

| Domestic |  |  |  | 
|---|---|---|---|
| Market | Release Date | Opening | Gross | 
| [Domestic](https://www.boxofficemojo.com/release/rl170295297/?ref_=bo_gr_rls) | Jul 17, 2026 | $123,502,900 | $620,585,090 | 

> [Truncated: 10360 -> 383 chars, 3% retained]
[10/07/26 19:52:54] INFO     Processing request of type           server.py:733
                             CallToolRequest                                   
[10/07/26 19:52:55] INFO     HTTP Request: GET                  _client.py:1740
                             https://www.boxofficemojo.com/rele                
                             asegroup/gr594236165/ "HTTP/1.1                   
                             200 OK"                                           
                    INFO     Processing request of type           server.py:733
                             ListToolsRequest                                  
MiniAgent Acting >mcp__web_read的执行结果：## Toy Story 5 - Box Office Mojo

> URL: https://www.boxofficemojo.com/releasegroup/gr594236165/ | Method: trafilatura | Quality: 68% (density: 92%, structure: 1%, clean: 100%, complete: 70%) | cache:MISS

# Toy Story 5

Woody, Buzz, Jessie and the rest of the gang's jobs are challenged when they're introduced to electronics, a new threat to playtime.

| Domestic |  |  |  | 
|---|---|---|---|
| Market | Release Date | Opening | Gross | 
| [Domestic](https://www.boxofficemojo.com/release/rl3344859137/?ref_=bo_gr_rls) | Jun 19, 2026 | $159,677,837 | $480,294,461 | 

> [Truncated: 6536 -> 363 chars, 5% retained]
[10/07/26 19:52:59] INFO     Processing request of type           server.py:733
                             CallToolRequest                                   
[10/07/26 19:53:00] INFO     HTTP Request: GET                  _client.py:1740
                             https://www.boxofficemojo.com/rele                
                             asegroup/gr3549843973/ "HTTP/1.1                  
                             200 OK"                                           
[10/07/26 19:53:01] INFO     Processing request of type           server.py:733
                             ListToolsRequest                                  
MiniAgent Acting >mcp__web_read的执行结果：## Michael - Box Office Mojo

> URL: https://www.boxofficemojo.com/releasegroup/gr3549843973/ | Method: trafilatura | Quality: 68% (density: 92%, structure: 1%, clean: 100%, complete: 70%) | cache:MISS

# Michael

The early life of musician Michael Jackson, from the discovery of his talent as the lead of the Jackson Five to the artist whose creative ambition fueled a pursuit to become the biggest entertainer in the world.

| Domestic |  |  |  | 
|---|---|---|---|
| Market | Release Date | Opening | Gross | 
| [Domestic](https://www.boxofficemojo.com/release/rl1246593025/?ref_=bo_gr_rls) | Apr 24, 2026 | $97,206,874 | $372,303,388 | 

> [Truncated: 10129 -> 437 chars, 4% retained]
[10/07/26 19:53:04] INFO     Processing request of type           server.py:733
                             CallToolRequest                                   
[10/07/26 19:53:05] INFO     HTTP Request: GET                  _client.py:1740
                             https://www.boxofficemojo.com/rele                
                             asegroup/gr513036805/ "HTTP/1.1                   
                             200 OK"                                           
[10/07/26 19:53:06] INFO     Processing request of type           server.py:733
                             ListToolsRequest                                  
MiniAgent Acting >mcp__web_read的执行结果：## The Super Mario Galaxy Movie - Box Office Mojo

> URL: https://www.boxofficemojo.com/releasegroup/gr513036805/ | Method: trafilatura | Quality: 68% (density: 92%, structure: 1%, clean: 100%, complete: 70%) | cache:MISS

# The Super Mario Galaxy Movie

Mario ventures into space, exploring cosmic worlds and tackling galactic challenges far from the familiar Mushroom Kingdom.

| Domestic |  |  |  | 
|---|---|---|---|
| Market | Release Date | Opening | Gross | 
| [Domestic](https://www.boxofficemojo.com/release/rl3344662529/?ref_=bo_gr_rls) | Apr 1, 2026 | $131,703,340 | $429,814,610 | 

> [Truncated: 9917 -> 370 chars, 3% retained]
这是第 4 次循环...
MiniAgent observation ...
MiniAgent> 

## 2026 年度全球累计票房榜 TOP 5

数据来源：Box Office Mojo「2026 Worldwide Box Office」（https://www.boxofficemojo.com/year/world/2026/），数据截至约北京时间 2026 年 10 月 7 日；上映时间取自各片 Release Group 详情页的北美（Domestic）上映日。

| 排名 | 片名 | 北美上映时间 | 全球累计票房 | 北美票房 | 海外票房 |
|---|---|---|---|---|---|
| 1 | Spider-Man: Brand New Day（蜘蛛侠：全新一天） | 2026-07-31 | **$2,506,251,053** | $954,734,627（38.1%） | $1,551,516,426（61.9%） |
| 2 | The Odyssey（奥德赛） | 2026-07-17 | **$1,768,084,090** | $620,585,090（35.1%） | $1,147,499,000（64.9%） |
| 3 | Toy Story 5（玩具总动员5） | 2026-06-19 | **$1,148,032,461** | $480,294,461（41.8%） | $667,738,000（58.2%） |
| 4 | Michael（迈克尔） | 2026-04-24 | **$1,028,703,388** | $372,303,388（36.2%） | $656,400,000（63.8%） |
| 5 | The Super Mario Galaxy Movie（超级马力欧银河大电影） | 2026-04-01 | **$1,012,578,077** | $429,814,610（42.4%） | $582,763,467（57.6%） |

补充亮点：榜首《Spider-Man: Brand New Day》开画北美首周末即达 $360,091,572；第 2 名《The Odyssey》首周末 $123,502,900；第 5 名《超级马力欧银河大电影》首周末 $131,703,340，与第 4 名《Michael》仅差约 1600 万美元，位次仍有变动可能。

**总结**：全程仅使用 MCP 网页工具（web_search 定位 → web_read 抓取 Box Office Mojo 的 2026 全球年度榜页面，再逐一抓取前 5 名的 release group 详情页补齐上映日期），未使用 bash/curl；已确认页面数据完整并交叉核对票房与详情页 gross 一致，最终输出上表所示 2026 年度全球票房前 5 名的名称、上映时间与票房。
```

4.故意在MCP Server中使用print，观察json解析报错。

> 添加前

```
你> 使用mcp工具查询今天日期
这是第 1 次循环...
MiniAgent observation ...
MiniAgent Reasoning > 用户要求使用 MCP 工具查询今天日期。直接调用 mcp__current_time。

[10/07/26 19:59:36] INFO     Processing request of type           server.py:733
                             CallToolRequest                                   
                    INFO     Processing request of type           server.py:733
                             ListToolsRequest                                  
MiniAgent Acting >mcp__current_time的执行结果：2026-10-07 19:59:36
这是第 2 次循环...
MiniAgent observation ...
MiniAgent Reasoning > User just asked for today's date. Maybe also format it. Let me provide it.

I could also call time_format to convert. Let's do that for a nicer display.

[10/07/26 19:59:42] INFO     Processing request of type           server.py:733
                             CallToolRequest                                   
                    INFO     Processing request of type           server.py:733
                             ListToolsRequest                                  
MiniAgent Acting >mcp__time_format的执行结果：2026年10月07日 19时59分36秒
这是第 3 次循环...
MiniAgent observation ...
MiniAgent> 

已通过 MCP 工具查询并格式化完成：

- **原始时间戳**：2026-10-07 19:59:36
- **格式化日期**：2026年10月07日
```

> 添加后

```
你> 使用mcp工具查询今天的日期
这是第 1 次循环...
MiniAgent observation ...
MiniAgent Reasoning > 用户要求使用 MCP 工具查询今天的日期。我应该调用 mcp__current_time 工具。

[10/07/26 20:13:24] INFO     Processing request of type           server.py:733
                             CallToolRequest                                   
MiniAgent Acting >mcp__current_time的执行结果：错误：MCP 工具调用失败：ExceptionGroup: unhandled errors in a TaskGroup (1 sub-exception)
```

添加时发现的问题

只写`print("【日志】: 开始执行日期转换函数")`会存入缓冲区，没有进入管道，无法触发协议污染。

写`print("【日志】: 开始执行日期转换函数", flush=True)`强制清空 stdout 缓冲区，直接把文本丢进管道，触发报错。

```
你> 使用mcp工具查询今天的日期
这是第 1 次循环...
MiniAgent observation ...
MiniAgent Reasoning > 用户要求使用 MCP 工具查询今天的日期。我应该调用 mcp__current_time 工具。

[10/07/26 20:13:24] INFO     Processing request of type           server.py:733
                             CallToolRequest                                   
MiniAgent Acting >mcp__current_time的执行结果：错误：MCP 工具调用失败：ExceptionGroup: unhandled errors in a TaskGroup (1 sub-exception)
```

## 代码

```python
from openai import OpenAI
import os
import sys
from pathlib import Path
import json
from session import Session
from compact_prompt import COMPACT_PROMPT
import re
from mcp_client import _list_tools, _call_tool
import asyncio

SYSTEM_PROMPT = "你是 MiniAgent，一个能独立完成编码任务的命令行 Agent。工作方式：先观察（读文件、跑命令），再动手（改文件），" \
                "改完必须验证（重新运行测试或程序），确认无误后用一段话总结你做了什么。不要凭想象修改没读过的文件。"

MODEL = "Qwen3.8-Flash-Next"
# 改成你对应服务商的base_url
BASE_URL = "https://api.hcnsec.cn/v1"
MAX_AGENT_LOOP = 20
COMPACT_THRESHOLD = 50000
KEEP_RECENT = 5
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
    },
    {
        "type": "function",
        "function": {
            "name": "grep_code",
            "description": "递归扫描目录内文本文件，使用正则搜索匹配行，最多返回50条结果，自动跳过隐藏目录与二进制文件, 如果存在多个文件应该先 grep 定位再 read_file，不要逐个文件读。",
            "parameters": {
                "type": "object",
                "properties": {
                    "pattern": {
                        "type": "string",
                        "description": "正则匹配表达式"
                    },
                    "directory": {
                        "type": "string",
                        "description": "检索根目录，默认当前目录 .",
                        "default": "."
                    },
                    "file_glob": {
                        "type": "string",
                        "description": "文件glob匹配规则，例如 *.py",
                        "default": "*"
                    }
                },
                "required": ["pattern"],
                "additionalProperties": False
            }
        }
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
    # return "失败"


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
    print(f"执行了bash命令：{command}")
    try:
        result = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=10)
        return f"输出是：stdout:\n{result.stdout}\n错误是：stderr:\n{result.stderr}"
    except Exception as e:
        return f"run_bash error: {str(e)}"
    # return "失败"


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


def grep_code(pattern: str, directory: str = ".", file_glob: str = "*") -> str:
    print(f"执行了grep_code: pattern={pattern}, directory={directory}, file_glob={file_glob}")
    regex = re.compile(pattern)
    hits = []
    for path in sorted(Path(directory).rglob(file_glob)):
        if not path.is_file() or any(p.startswith(".") for p in path.parts):
            continue                      # 跳过隐藏目录（.git、.venv）
        try:
            lines = path.read_text().splitlines()
        except (UnicodeDecodeError, OSError):
            continue                      # 跳过二进制
        for lineno, line in enumerate(lines, 1):
            if regex.search(line):
                hits.append(f"{path}:{lineno}: {line.strip()[:200]}")
                if len(hits) >= 50:
                    hits.append("...（超过 50 条命中，请用更精确的 pattern）")
                    return "\n".join(hits)
    return "\n".join(hits) if hits else "没有找到匹配"


# 工具路由表：函数名 → 本地真实函数
TOOL_FUNCTIONS = {
    "read_file": read_file,
    "write_file": write_file,
    "edit_file": edit_file,
    "run_bash": run_bash,
    "list_files": list_files,
    "grep_code": grep_code,
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


def append_message(session, message, messages: list[dict]):
    """向对话历史追加一条消息，并保存到文件"""
    session.append(message)
    messages.append(message)
    return messages


def current_size(messages: list[dict]) -> int:
    """预估当前对话历史大小（token），三个字符算一个token"""
    total_chars = 0
    for msg in messages:
        total_chars += len(msg.get("role", ""))
        total_chars += len(msg.get("content", ""))
    # 等价向上取整 (a + b -1) // b
    return (total_chars + 3 - 1) // 3


def compact_history(messages: list[dict], client) -> list[dict]:
    """压缩对话历史，只保留最近的几条消息"""
    cut = len(messages) - KEEP_RECENT
    if cut <= 1:
        history_str = json.dumps(messages[1:], ensure_ascii=False)
        recent = []
    else:
        # 不能把 tool 消息和它的 assistant 请求切开，往后挪到安全边界
        while cut < len(messages) and messages[cut]["role"] == "tool":
            cut += 1
        old, recent = messages[1:cut], messages[cut:]
        history_str = json.dumps(old, ensure_ascii=False)
    compact_content = COMPACT_PROMPT + history_str
    response = client.chat.completions.create(
        model=MODEL,
        messages=[{"role": "user", "content": compact_content}],
    )
    summary = response.choices[0].message.content
    return [messages[0],  # system 原样
            {"role": "user", "content": f"[此前工作的摘要]\n{summary}"},
            *recent]

def main() -> None:
    # 用工具查一下现在几点，再统计./code_data目录下 Python 代码规模，一起汇报。
    api_key = load_api_key()
    client = OpenAI(api_key=api_key, base_url=BASE_URL)
    session = Session.get()
    # ① 启动时：发现 MCP 工具，和本地工具合并成一张表
    mcp_schemas = asyncio.run(_list_tools())
    all_schemas = TOOL_SCHEMAS + mcp_schemas
    messages = []
    append_message(session, {"role": "system", "content": SYSTEM_PROMPT}, messages)

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

        append_message(session, {"role": "user", "content": user_input}, messages)

        current_loop = 1
        reply = ""
        while current_loop <= MAX_AGENT_LOOP:
            print(f"这是第 {current_loop} 次循环...")
            compact_before = current_size(messages)
            if compact_before > COMPACT_THRESHOLD:
                print("对话历史过长，正在压缩...")
                messages = compact_history(messages, client)
                session.rewrite(messages)
                compact_after = current_size(messages)
                print("压缩完成:" + str(compact_before), "token -> ", str(compact_after) + "token")

            current_loop = current_loop + 1
            print("MiniAgent observation ...")
            response = client.chat.completions.create(
                model=MODEL,
                messages=messages,
                tools=all_schemas,
                tool_choice="auto"
            )
            message = response.choices[0].message
            if not message.tool_calls:  # 没想调工具，直接说了答案
                reply = message.content
                break
            # 保存工具调用信息
            append_message(session,
                           {
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
                           }
            , messages)
            print("MiniAgent Reasoning > " + response.choices[0].message.reasoning)
            # 执行工具调用
            for call in message.tool_calls:
                if call.function.name.startswith("mcp_"):
                    try:
                        result = asyncio.run( _call_tool(call.function.name.split("__")[1], json.loads(call.function.arguments)))
                    except Exception as exc:
                        result = f"错误：MCP 工具调用失败：{type(exc).__name__}: {exc}"
                else:
                    result = execute_tool_call(call)
                print("MiniAgent Acting >" + call.function.name + "的执行结果：" + result)
                append_message(session, {
                    "role": "tool",
                    "tool_call_id": call.id,
                    "content": result
                }, messages)

        if reply == "" or current_loop > MAX_AGENT_LOOP:
            response = client.chat.completions.create(
                model=MODEL,
                messages=messages
            )
            reply = response.choices[0].message.content
        print("MiniAgent> " + reply)
        # 追加assistant回复，维持上下文记忆
        append_message(session, {"role": "assistant", "content": reply}, messages)


if __name__ == "__main__":
    main()

```

