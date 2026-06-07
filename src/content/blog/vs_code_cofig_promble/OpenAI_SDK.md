---
title: OpenAI SDK
description: 关于OpenAI SDK接入DeepSeek的一些注意事项，犯的错误
publishDate: 2026-06-04
tags:
  - 学习
language: 中文
draft: false
---



# OpenAI SDK

这是我Agent学习的始发站,在这里主要学习三点：

- 多轮对话和上下文管理，流式输出
- 结构化输出
- 工具/函数调用



### 一.Chat Completions

这里是用while循环写的一个终端多对话脚本，使用流式输出

```python
import os
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()

client = OpenAI(
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com"
)

history = []

while True:
    user_input = input("请输入问题（输入 'exit' 退出）：")
    if user_input.lower() == 'exit':
        print("退出程序。")
        break
    history.append({"role": "user", "content": user_input})

    response = client.chat.completions.create(
    model="deepseek-v4-flash",
    messages = history,
    stream = True
    )

    full_response_content = ""
    for chunk in response:
        content = chunk.choices[0].delta.content
        if content is not None:
            print(content,end="",flush = True)
            full_response_content += content
    print() # 流式打印后换行        
    history += [{"role": "assistant", "content": full_response_content}]
    

```

因为deepseek不支持Response API，所以相较于OpenAI的原生模型，写起来要繁琐一些，需要手动管理history，这里主要有两点值得注意：

1.**history管理:**assistant的对话一定要append进去，否则每一次他会对之前的所有问题都进行回答

2.**流式输出和普通输出的区别**:response里加strea=True,在输出时，普通输出只需要print(response.choices[0].message.content),但在流式输出时，response是一段一段的，它是按token输出(delta的用处)，会变化。所以这里我们手动装填，就相当于普通情况下的输出(默认当缓冲区满时输出，即flush=false)



### 二.Structured Outputs

还是因为deepseek不是OpenAI的原生模型，这里在自己写时需要变动。OpenAI SDK原文档中的parse我们是不能用的

```python
import os
from openai import OpenAI
from dotenv import load_dotenv
from pydantic import BaseModel
import json

load_dotenv()

client = OpenAI(
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com"
)

class CalendarEvent(BaseModel):
    name: str
    date: str
    participants: list[str]

response = client.chat.completions.create(
    model="deepseek-v4-flash",
    messages=[
        {"role":"system","content":"请提取活动信息，并且只返回 JSON。输出格式为 {\"name\": ..., \"date\": ..., \"participants\": [...]}"},
        {"role":"user","content":"Alice和Bob周五准备去科技展。"},
    ],
    response_format={"type":"json_object"}
)

json_string = response.choices[0].message.content

event_data = json.loads(json_string)
event = CalendarEvent(**event_data)

print(event.name)
print(event)
```

这里有一个点需要注意：

- 我们在进行结构化输出json时，系统提示词（System Prompt）或用户提示词中，必须显式包含 “JSON” 这个单词！也就是说，必须要把格式说的很明白才行

  ```python
  {"role":"system","content":"请提取活动信息，并且只返回 JSON。"}
  ```

  这样也是会报错的



### 三.Function Calling

```python
import os
import json
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()

client = OpenAI(
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com"
)

# 1. 模拟的天气数据库/API
def get_weather(location: str) -> str:
    weather_data = {
        "北京": "晴，28°C",
        "上海": "多云，26°C",
        "杭州": "小雨，24°C",
        "广州": "雷阵雨，32°C",
    }
    return weather_data.get(location, f"{location}天气未知，25°C")

# 2. 工具说明书
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取某个城市的天气信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "用户要查询天气的城市，例如 北京、上海、杭州"
                    }
                },
                "required": ["location"]
            }
        }
    }
]

# 3. 初始化对话历史
messages = [
    {"role": "system", "content": "你是一个天气助手。当用户问天气时，优先调用 get_weather 工具。"}
]

print("☀️ 天气助手已上线！（输入 'exit' 或 'quit' 退出）")

# 4. 开启循环聊天
while True:
    # 让用户在终端动态输入问题
    user_input = input("\n我：")
    if user_input.strip().lower() in ['exit', 'quit']:
        print("助手：再见！")
        break
        
    if not user_input.strip():
        continue

    # 把用户输入的话加入到对话历史中
    messages.append({"role": "user", "content": user_input})

    # 第一次请求：让大模型判断是否需要调用工具
    response = client.chat.completions.create(
        model="deepseek-v4-flash",
        messages=messages,
        tools=tools,
        tool_choice="auto"
    )

    message = response.choices[0].message

    # 检查大模型是否触发了工具调用
    if message.tool_calls:
        tool_call = message.tool_calls[0]
        tool_name = tool_call.function.name
        tool_args = json.loads(tool_call.function.arguments)

        if tool_name == "get_weather":
            # 提取大模型锁定的城市（比如：你输入“深圳天气如何”，它会自动提取出 “深圳”）
            location = tool_args["location"]
            print(f"🎬 [系统提示：大模型决定调用本地工具查询 -> {location}]")
            
            # 本地运行函数获取结果
            tool_result = get_weather(location)

            # 把大模型的“调用意向”和“本地查询结果”都塞进历史记录
            messages.append(message)
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": tool_result
            })

            # 第二次请求：大模型拿到真实天气数据，组织语言回复用户
            final_response = client.chat.completions.create(
                model="deepseek-v4-flash",
                messages=messages,
                tools=tools
            )

            assistant_reply = final_response.choices[0].message.content
            print(f"助手：{assistant_reply}")
            
            # 把 AI 最终的完整回答也存入历史，以便下次多轮对话
            messages.append({"role": "assistant", "content": assistant_reply})
            
    else:
        # 如果你问的不是天气（例如输入“你好”），模型不会触发工具，直接走这里回复
        assistant_reply = message.content
        print(f"助手：{assistant_reply}")
        messages.append({"role": "assistant", "content": assistant_reply})
```

理解机制：

