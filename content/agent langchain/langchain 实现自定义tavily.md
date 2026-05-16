```json
{
  "date": "2026.04.13 23:01",
  "tags": ["LangChain", "AI Agent", "tavily"],
  "description": "通过自定义tavily，减少默认情况下网页搜索消耗的token，并把信息源添加到返回结果"
}
```

### 1. tavily使用

LangChain中提供了tavily用来做web搜索，通常使用方式如下
```jupyter
from langchain_tavily import TavilySearch

search_tool = TavilySearch(
    max_results=5,
    topic="general", # general, news, finance
    # include_answer=False,
    # include_raw_content=False,
    # include_images=False,
    # include_image_descriptions=False,
    # search_depth="basic",
    # time_range="day",
    # include_domains=None,
    # exclude_domains=None
)

agent = create_agent(
    model="deepseek-chat",
    tools=[search_tool],
    system_prompt="你是一个智能助手，你使用工具来解决用户问题。"
)
response = agent.invoke(
    {"messages": [HumanMessage(content="杨柳絮为什么那么多？")]},
)

for message in response['messages']:
    message.pretty_print()

```

### 2. tavily常见问题

目前的搜索智能体存在两个问题：
- 官方默认的tavily工具过于复杂，包含了完整的参数列表，会导致额外的流量和Token消耗
- 结果中不包含网页数据源，可信度低

解决思路：
- 自定义tavily工具
- 结构化输出

### 3. 自定义tavily工具

代码实现如下，首先是把tavily封装成tool
```jupyter
# 先使用官方的客户端做初始化
tavily = TavilySearch(
    max_results=5,
    topic="general"
)

# 然后自己封装为tool
@tool
def web_search(query: str):
    """Search the web for information"""
    return tavily.invoke(query)
```

自定义输出的结构体
```jupyter
from pydantic import BaseModel, Field

# Agent回答内容引用的网页信息
class Reference(BaseModel):
    title: str = Field(description="The title of the web page cited in the answer")
    url: str = Field(description="The url of the web page cited in the answer")

# Agent的回答内容
class AnswerInfo (BaseModel):
    answer: str = Field(description="The final answer for user")
    reference: list[Reference] = Field(description="The web pages cited in the answer")
```

创建agent调用tavily
```jupyter
agent = create_agent(
    model="deepseek-chat",
    tools=[web_search],
    system_prompt="你是一个智能助手，你使用工具来解决用户问题。",
    response_format=AnswerInfo
)
#%%
# 调用agent
response = agent.invoke(
    {"messages": [HumanMessage(content="杨柳絮为什么那么多？")]},
)

# 获取结构化输出
print(response['structured_response'])
```