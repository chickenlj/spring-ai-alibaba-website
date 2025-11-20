---
title: 快速开始
description: 学习如何使用 Spring AI Alibaba Studio 可视化的与 Agent 通信
keywords: [Studio, Agent Chat, UI, Spring AI Alibaba]
---

# 快速开始

Agent Chat UI 为开发者提供了一种可视化的方式，可以与任何 Spring AI Alibaba 开发的 Agent 进行聊天。

![Agent Chat UI](/img/chatui/agent-chat-ui.gif)

## 快速体验

社区在 <a href="https://github.com/alibaba/spring-ai-alibaba/tree/main/examples/chatbot" target="_blank">examples/chatbot</a>
提供了一个可快速运行的 ChatBot 智能体示例，支持 Python 脚本、Shell脚本、查看本地文件等工具调用。

### 前置条件

* JDK 17+
* Maven 3.8+
* 选择你的 LLM 提供商并获取 API-KEY（如阿里云百炼的 DashScope）

### 下载示例

示例代码在 spring-ai-alibaba 主干仓库的 examples 目录下：

```shell
git clone https://github.com/alibaba/spring-ai-alibaba.git
cd examples/chatbot
```

### 启动ChatBot智能体

运行以下命令，启动智能体:

```shell
mvn spring-boot:run
```

### 与智能体聊天（测试智能体）

示例启动后，打开浏览器访问 http://localhost:8080/chatui/index.html 页面即可与智能体聊天了，可以看到详细的工具调用、推理过程：

![agent-chat-ui](/img/agent/agents/chatbot-agent-chat-ui.gif)

## Agent Chat UI 工作原理

### 嵌入式模式

UI 可以在嵌入式模式下与任何 Spring Boot 应用程序一起工作。

只需在您的 agent 项目中添加以下依赖：

```xml
<dependency>
	<groupId>com.alibaba.cloud.ai</groupId>
	<artifactId>spring-ai-alibaba-studio</artifactId>
	<version>1.1.0.0-M5</version>
</dependency>
```

运行您的 agent，访问 `http:localhost:{your-port}/chatui/index.html`，现在您就可以与您的 agent 聊天了。

### 独立模式

首先，克隆仓库，

```bash
git clone https://github.com/alibaba/spring-ai-alibaba.git

cd spring-ai-alibaba/spring-ai-alibaba-studio/agent-chat-ui
```

安装依赖：

```bash
pnpm install
# or
# npm install
```

运行应用：

```bash
pnpm dev
# or
# npm run dev
```

应用将在 `http://localhost:3000` 可用。

默认情况下，UI 连接到您的后端 Agent，地址为 `http://localhost:8080`，您可以在 `.env.development` 文件中更改地址。

```properties
# .env.development
NEXT_PUBLIC_API_URL=http://localhost:8080
# The agent to call in the backend application, backend application should register agent as required, check examples for how to configure.
NEXT_PUBLIC_APP_NAME=research_agent
NEXT_PUBLIC_USER_ID=user-001
```

