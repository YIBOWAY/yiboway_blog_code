---
title: 如何利用ChatGPT构建一个股票筛选助手
date: 2023-06-11 07:37:31
tags:
  - ChatGPT
  - stock
categories: ChatGPT
description: 这篇文章是关于如何使用Financial Modeling Prep API构建一个ChatGPT插件，该插件可以作为股票筛选助手，根据用户的查询来检索和筛选股票。文章中详细介绍了如何创建和部署这个插件。
cover: https://yiboway-blog-1301945320.cos.ap-hongkong.myqcloud.com/img/blog_article/chatgpt-plugin-stock-screener.png
katex: true
---

# 如何利用ChatGPT构建一个股票筛选助手插件？

本篇文章为翻译版本，原文链接可点击[这里](https://www.mlq.ai/chatgpt-plugins-stock-screener-assistant/)。

在我们之前的ChatGPT插件教程中，我们已经介绍了如何设置一个简单的待办事项列表并构建一个AI新闻助手。在这篇指南中，我们将构建一个新的ChatGPT插件，它可以作为一个股票筛选助手，也就是说，它可以根据用户的查询来检索和筛选股票。为了做到这一点，我们将使用Financial Modeling Prep API，特别是股票筛选器端点，并将在Replit上托管代码。为了做到这一点，我们需要构建这个插件的三个主要文件包括：

+ 一个main.py文件，用于提供API的功能
+ 一个插件清单ai-plugin.json文件，用于提供元数据
+ 一个openapi.yaml文件，用于为ChatGPT文档化API

例如，有了这个插件，我可以询问这样的问题: 

*展示给我市值超过10亿美元，每股股息超过2美元的科技公司*

![](https://yiboway-blog-1301945320.cos.ap-hongkong.myqcloud.com/img/blog_article/Screenshot-2023-04-26-at-5.12.04-PM.png)

如你所见，股票筛选器允许用户指定他们的标准，如：

+ 市值
+ 部门和行业
+ 股息
+ Beta值，等等

好的，现在我们知道我们要构建什么，让我们开始构建API功能。

# 第一步：构建API功能

第一步，我们将创建一个新的Python Repl，并创建一个main.py文件来提供我们API的功能和我们需要的其他文件。

## Imports and constants

首先，我们将导入必要的包，初始化我们的Flask应用，并定义两个Financial Modeling Prep API的常量：
```python
import os
import logging
from logging import StreamHandler
from waitress import serve
from flask import Flask, request, jsonify, send_from_directory
import requests

app = Flask(__name__)

FMP_API_KEY = "your-api-key"
FMP_API_URL = "https://financialmodelingprep.com/api/v3"
```

## 获取股票筛选数据

接下来，我们将定义一个股票筛选器函数，该函数接收用户定义的查询参数，并向Financial Modeling Prep发送请求：
```python
def stock_screener(**params):
  url = f"{FMP_API_URL}/stock-screener"
  params["apikey"] = FMP_API_KEY
  response = requests.get(url, params=params)
  if response.status_code == 200:
    return response.json()
  else:
    raise Exception(f"Error: {response.status_code}, {response.text}")
```

## 获取股票筛选器数据的API路由

接下来，我们将创建一个API路由，用于处理传入的用户定义的查询参数，调用我们刚刚创建的股票筛选器函数，并以JSON格式返回结果：
```python
@app.route('/stock_screener', methods=['GET'])
def fetch_stock_screener():
  app.logger.info("Fetching Stock Screener")
  query_params = request.args.to_dict()
  screener_data = stock_screener(**query_params)
  return jsonify(screener_data)
```

## 提供插件文件

现在我们已经有了我们的Financial Modeling Prep API函数，我们需要三个更多的路由来提供必要的ChatGPT插件文件，包括ai-plugin.json、openapi.yaml和插件logo：
```python
@app.route('/.well-known/ai-plugin.json', methods=['GET'])
def send_ai_plugin_json():
  return send_from_directory('.well-known', 'ai-plugin.json')
```

## 提供openapi.yaml

接下来，我们需要一个路由来提供openapi.yaml文件，这个文件被ChatGPT用来理解插件可以做什么以及如何与API端点交互：
```python
@app.route('/openapi.yaml', methods=['GET'])
def send_openapi_yaml():
  return send_from_directory('.', 'openapi.yaml')
```

## 提供插件logo

最后，我们将创建一个路由来提供插件logo：
```python
@app.route('/logo.png', methods=['GET'])
def send_logo():
  return send_from_directory('.', 'logo.png')
```

有了这些API函数和路由，我们的Flask应用现在已经设置好来处理来自ChatGPT的传入请求并提供必要的文件。
```python
if __name__ == "__main__":
  app.run(host='0.0.0.0', port=int(os.environ.get('PORT', 8080)))
```

# 第二步：创建插件清单

接下来，让我们创建插件清单文件，该文件：

+ 包括关于你的插件的元数据（名称、logo等）
+ 关于需要的认证的详细信息（认证类型、OAuth URLs等）
+ 你想要公开的端点的OpenAPI规范

对于这个插件，我们将在我们的清单中包括以下信息：

+ 同时方便人类和模型理解的名称和描述
+ 用户认证，对于这个例子，我们只设置为none
+ API配置，其中包括你需要用你的Replit URL更新的OpenAPI URL
+ Logo URL和联系信息
  
```json
{
  "schema_version": "v1",
  "name_for_human": "Stock Screener Assistant",
  "name_for_model": "stockScreener",
  "description_for_human": "Plugin for fetching stocks based on various criteria like market capitalization, price, beta, volume, dividend, etc.",
  "description_for_model": "Plugin for fetching stocks based on various criteria like market capitalization, price, beta, volume, dividend, etc. Help users find stocks that meet their requirements.",
  "auth": {
    "type": "none"
  },
  "api": {
    "type": "openapi",
    "url": "https://your-repl-url.repl.co/openapi.yaml",
    "is_user_authenticated": false
  },
  "logo_url": "https://your-repl-url.repl.co/logo.png",
  "contact_email": "support@example.com",
  "legal_info_url": "http://www.example.com/legal"
}
```

如上所述，你只需要用你的Repl的实际URL更新your-repl-url。在创建ai-plugin.json文件后，你还需要将它保存在.well-known文件夹中，以便ChatGPT使用它。

有了我们的插件清单文件，我们需要的最后一件事就是用OpenAPI来文档化我们的API。

# 第三步：用OpenAPI文档化API

在这一步，我们需要创建一个OpenAPI规范，正如OpenAI所强调的：

模型将看到OpenAPI描述字段，这些字段可以用来为不同的字段提供自然语言描述。

在这种情况下，你需要更新的所有内容就是这里的服务器URL：https://your-repl-url.username.repl.co。

然后，OpenAPI规范定义了/stock_screener端点和用户可以自定义的参数，包括市值、价格、贝塔、股息等。最后，组件部分定义了JSON响应的模式：
```json
openapi: "3.0.2"
info:
  title: "Stock Screener Assistant API"
  version: "1.0.0"
servers:
  - url: "https://stockscreenerplugin.peterfoy2.repl.co"
paths:
  /stock_screener:
    get:
      operationId: fetchStockScreener
      summary: "Fetch Stock Screener"
      parameters:
        - in: query
          name: marketCapMoreThan
          schema:
            type: number
          required: false
          description: "Market capitalization greater than the specified value."
        - in: query
          name: marketCapLowerThan
          schema:
            type: number
          required: false
          description: "Market capitalization lower than the specified value."
        - in: query
          name: priceMoreThan
          schema:
            type: number
          required: false
          description: "Price greater than the specified value."
        - in: query
          name: priceLowerThan
          schema:
            type: number
          required: false
          description: "Price lower than the specified value."
        - in: query
          name: betaMoreThan
          schema:
            type: number
          required: false
          description: "Beta greater than the specified value."
        - in: query
          name: betaLowerThan
          schema:
            type: number
          required: false
          description: "Beta lower than the specified value."
        - in: query
          name: volumeMoreThan
          schema:
            type: number
          required: false
          description: "Volume greater than the specified value."
        - in: query
          name: volumeLowerThan
          schema:
            type: number
          required: false
          description: "Volume lower than the specified value."
        - in: query
          name: dividendMoreThan
          schema:
            type: number
          required: false
          description: "Dividend greater than the specified value."
        - in: query
          name: dividendLowerThan
          schema:
            type: number
          required: false
          description: "Dividend lower than the specified value."
        - in: query
          name: isEtf
          schema:
            type: boolean
          required: false
          description: "Is ETF (true/false)."
        - in: query
          name: isActivelyTrading
          schema:
            type: boolean
          required: false
          description: "Is actively trading (true/false)."
        - in: query
          name: sector
          schema:
            type: string
          required: false
          description: "Sector (e.g., Technology, Healthcare, etc.)."
        - in: query
          name: industry
          schema:
            type: string
          required: false
          description: "Industry (e.g., Software, Banks, etc.)."
        - in: query
          name: country
          schema:
            type: string
          required: false
          description: "Country (e.g., US, UK, CA, etc.)."
        - in: query
          name: exchange
          schema:
            type: string
          required: false
          description: "Exchange (e.g., NYSE, NASDAQ, etc.)."
        - in: query
          name: limit
          schema:
            type: integer
          required: false
          description: "Limit the number of results."
      responses:
        "200":
          description: "A list of stocks matching the given criteria."
          content:
            application/json:
              schema:
                type: array
                items:
                  type: object
                  properties:
                    symbol:
                      type: string
                    companyName:
                      type: string
                    marketCap:
                      type: number
                    sector:
                      type: string
                    beta:
                      type: number
                    price:
                      type: number
                    lastAnnualDividend:
                      type: number
                    volume:
                      type: number
                    exchange:
                      type: string
                    exchangeShortName:
                      type: string

```

# 第四步：部署和测试ChatGPT插件

有了这三个文件，我们现在可以去部署和测试新的ChatGPT插件。我们需要做的是：

+ 在你向Replit添加了所有必要的文件后，点击"Run"来启动服务器并使插件可用
+ 前往ChatGPT并导航到插件商店
+ 选择"Develop your own plugin"并输入你的Replit URL
![](https://yiboway-blog-1301945320.cos.ap-hongkong.myqcloud.com/img/blog_article/Screenshot-2023-04-28-at-11.50.25-AM.png)

在安装和启用新插件后，你可以测试它并指定你的股票搜索条件，我也可以确认在我写这篇文章的时候苹果的市值是正确的：
![](https://yiboway-blog-1301945320.cos.ap-hongkong.myqcloud.com/img/blog_article/Screenshot-2023-04-28-at-11.56.02-AM.png)

# 总结：ChatGPT股票筛选器插件

在这篇指南中，我们看到了如何使用Financial Modeling Prep API构建一个简单的股票筛选ChatGPT插件。回顾一下，我们：

+ 在main.py文件中创建了一个Flask应用，创建必要的路由接口
+ 在ai-plugin.json清单文件中定义了我们插件的元数据
+ 使用OpenAPI文档化了API，概述了/stock_screener接口、参数和响应模式
+ 在Replit上托管了项目，并在插件商店中进行了测试

还有一些步骤我们想要包括在将这样的插件投入生产中，比如限制速率和用户认证，我们将在未来的教程中讨论这些。