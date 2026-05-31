---
title: VS Code中python的配置问题
description: python解释器，虚拟环境
publishDate: 2026-05-30
tags:
  - 学习
language: 中文
draft: true
---



#### 1.用py新建虚拟环境

`py -m venv .venv` 



#### 2.激活虚拟环境

`.\.venv\Scripts\Activate.ps1` 

每次打开vs code都需要重新激活



#### 3.安装需要的库

`python -m pip install requests` （这里用python）



#### 4.指定解释器

- 一种做法：Ctrl + Shift + P 后输入 Python: Select Interpreter 选择项目中的 .\\.venv\Scripts\python.exe

- 另一种做法：新建一个.vscode/settings.json文件，内容为： 

  ```json
  {
    "python.defaultInterpreterPath": "${workspaceFolder}\\.venv\\Scripts\\python.exe",
    "python.terminal.activateEnvironment": true
  }
  ```

  

#### 5.运行程序

`python your_file.py` 