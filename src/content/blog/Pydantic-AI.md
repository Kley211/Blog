---
title: Pydantic AI
description: 关于Pydantic AI 我遇到的一些问题
publishDate: 2026-06-07
tags:
  - 学习
language: 中文
draft: true
---





## Pydantic AI

- [ ] 同步异步共用一个agent实例

  ```
  问题现象
  
  在 agent_run.py 里，先用同一个 Agent 执行 run_sync()，然后再继续用它执行 run()、run_stream()、run_stream_events()，程序会卡在后续网络请求阶段，看起来像是“接口没返回”。
  
  问题原因
  
  根因不是 DeepSeek 网络坏了，也不是全局变量本身有问题，而是：
  
  run_sync() 本质上不是一套完全独立的“同步实现”，它其实是用事件循环把异步的 run() 包了一层同步调用。
  同时，OpenAIProvider / Agent 内部会持有异步客户端资源，比如 AsyncOpenAI 和 httpx.AsyncClient。
  
  这样一来，原来的代码就变成了：
  
  同一个 Agent 先被 run_sync() 使用了一次
  这次调用依赖的是一个事件循环
  后面又在 asyncio.run(async_main()) 里，用同一个 Agent 去执行异步方法
  asyncio.run(...) 会创建新的事件循环
  同一个 Agent 里缓存的异步资源被跨不同事件循环复用，容易出现阻塞、挂起或不可预期行为
  所以更准确地说：
  
  不是“全局 Agent 一定不能用”
  而是“同一个 Agent 实例不要混着跨同步包装和异步事件循环复用”
  解决方法
  
  最稳妥的做法是把同步和异步场景分开，让它们各自创建自己的 Agent 实例。
  
  例如：
  
  def build_agent() -> Agent:
      provider = OpenAIProvider(
          base_url="https://api.deepseek.com",
          api_key=os.getenv("DEEPSEEK_API_KEY"),
      )
      model = OpenAIChatModel(model_name="deepseek-v4-flash", provider=provider)
      return Agent(model=model)
  然后：
  
  同步函数里单独 agent = build_agent()
  异步函数里单独 agent = build_agent()
  这样就避免了同一个实例跨不同运行上下文复用。
  
  结论
  
  这个问题的本质是：
  
  同一个 pydantic_ai.Agent 实例被先同步调用、再异步调用，导致内部异步客户端和事件循环上下文混用，从而卡住。
  
  推荐做法是：
  
  要么全程都用异步
  要么同步和异步各自创建独立的 Agent
  不要让同一个 Agent 同时承担 run_sync() 和 asyncio.run(...) 里的异步调用
  ```

  

- [ ] 结构化输出不能使用推理模型

  ```
  这是一个非常典型的模型特性冲突报错！报错的核心原因在于这一行：
  
  Thinking mode does not support this tool_choice
  （思考模式不支持当前的工具选择策略）
  
  你使用的模型名是 deepseek-v4-flash（这是 DeepSeek 的 R1 推理/思考模型在某些第三方平台或特定版本下的映射名，其底层开启了深度思考（Thinking）模式）。
  
  🔍 为什么会报错？
  当你在 Pydantic AI 中设置了 output_type=CityLocation 时，框架为了确保大模型百分之百返回标准的 JSON 对象，会在后台默默对大模型下达一个死命令：
  
  “本次对话你必须且只能调用 FinalResult 这个格式化工具（即强制 tool_choice='required'）！”
  
  然而，DeepSeek 的深度思考（Thinking）模式有非常严格的限制：当模型在进行深度思考时，API 不允许用户强制指定 tool_choice。
  
  这就好比框架强迫 AI “必须立刻用左手拿筷子”，而 AI 的思考系统说“我正在闭眼打坐，现在不能用手”。两者冲突，DeepSeek 服务器直接愤怒地甩给你一个 400 BadRequestError。
  
  🛠️ 怎么解决？
  解决这个冲突有两种办法，你可以根据自己的需求二选一：
  
  办法 A：换用普通的聊天模型（推荐 🌟）
  如果你不需要它进行长篇大论的推理，只需要它精准提取城市和国家，应该换成普通的 DeepSeek-V3 模型。V3 模型完美支持强类型的结构化输出。
  
  把代码里的 model_name 改为：
  
  Python
  # 使用普通的 V3 聊天模型，它完美支持 output_type 强约束
  model = OpenAIChatModel(model_name="deepseek-chat", provider=provider)
  办法 B：如果你必须用这个推理模型，那就要把 output_type 改为文本
  因为思考模型不支持框架强制限制输出格式，你只能把 Agent 降级为普通的文本接收模式，然后靠提示词让它听话。
  
  Python
  # 1. 拿掉 output_type
  agent = Agent(
      model=model,
      system_prompt="请直接返回形如 {'city': '伦敦', 'country': '英国'} 的 JSON 字符串，不要包含任何 markdown 标记。"
  )
  
  # 2. 拿到纯文本后，自己在代码里手动用 Pydantic 解析
  result = agent.run_sync("2012年奥林匹克运动会在哪里举办的？")
  # 然后用 CityLocation.model_validate_json(result.output) 去解析
  💡 总结
  在玩 结构化输出（output_type） 时，切记不要使用带有 Thinking、Reasoning 或 R1 标签的深度思考模型，用普通的 deepseek-chat (V3) 或 gpt-4o-mini 才是最丝滑的选择！
  ```

  

- [ ] 