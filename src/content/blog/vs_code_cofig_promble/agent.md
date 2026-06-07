---
title: Agent学习
description: Agent入门
publishDate: 2026-06-01
tags:
  - 随笔
  - 学习
language: 中文
draft: false
---

# Agent学习



### 1.基础调用

学会创建 client、读 API key、发一次请求、拿到文本结果。

python路线：用 **OpenAI SDK + base_url 模式**接 DeepSeek。但是OpenAI SDK中现在推荐的是Responses API（仅支持自己家的模型），deepseek仍然还在使用Chat Completions API结构

```python
import os
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()

client = OpenAI(api_key=os.getnev(“DEEPSEEK_API_KEY”),base_url="https://api.deepseek.com")

response = client.chat.completions。create(
	model = "deepseek-v4-flash"，
	message = [
		{"role" : "user", "content":"你好"}
	]
)

print(response.choices[0].message.content)
```





### 2.上下文管理

学会什么是“历史消息”。OpenAI依旧是有只属于自家模型的机制，conversation和previous_response_id。

##### conversation:

逻辑符合日常使用gpt网页版，当你每次在网页端开启"New Chat"时，就会生成一个唯一的conversation_id。

不需要自己手动维护历史，只需要把conversation一并发过去

```python
# 第一轮：开启一个新会话（如果你不传这个参数，系统会自动为你生成一个并返回）
response1 = client.responses.create(
    model="gpt-4o-mini",
    input="我有一只猫，名字叫冒号。",
    conversation="my_cat_chat_001" # 自定义或使用系统返回的 ID
)

# 第二轮：直接问问题，不需要带上第一轮的对话
response2 = client.responses.create(
    model="gpt-4o-mini",
    input="它的名字叫什么来着？",
    conversation="my_cat_chat_001" # 认准同一个聊天室
)
print(response2.output_text) # 输出: "它的名字叫冒号。"
```



##### previous_response_id:

每一个Response在OpenAI系统中都有一个唯一的id。你想在哪个回答后面继续，就把那个回答的id作为previous_response_id传过去。灵感来源于Git的版本控制

```python
# 1. 提问
response_base = client.responses.create(
    model="gpt-4o-mini",
    input="请帮我写一首关于科技的诗。"
)
base_id = response_base.id # 拿到这个回答的身份证，假设是 "res_111"

# 分支 A：基于这个诗，要求换个风格
response_branch_a = client.responses.create(
    model="gpt-4o-mini",
    input="把它改成赛博朋克风格。",
    previous_response_id=base_id # 接在 res_111 后面
)

# 分支 B：同样基于原诗，要求翻译（它完全不受分支 A 的干扰！）
response_branch_b = client.responses.create(
    model="gpt-4o-mini",
    input="把它翻译成英文。",
    previous_response_id=base_id # 依然接在 res_111 后面
)
```



但是用deepseek的话只能手动管理了，后面Pydantic AI会解决这个问题的。

```python
from openai import OpenAI
import os
from dotenv import load_dotenv

load_dotenv() # 加载环境变量

client = OpenAI(api_key=os.getenv("DEEPSEEK_API_KEY"),base_url="https://api.deepseek.com")

history = [
    {
        "role": "user",
        "content": "讲个笑话"
    }
]

response = client.chat.completions.create(
    model = "deepseek-v4-flash",
    messages= history
)

first_reply = response.choices[0].message.content
print(first_reply)
history.append({"role":"assistant","content":first_reply})

history.append({"role":"user","content":"再讲一个笑话"})

second_response = client.chat.completions.create(
    model = "deepseek-v4-flash",
    messages= history
)

print(second_response.choices[0].message.content)
```

