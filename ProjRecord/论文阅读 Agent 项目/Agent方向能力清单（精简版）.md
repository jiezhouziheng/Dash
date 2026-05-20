# Agent 方向能力清单（精简版）

适用对象：准备 Agent/AIGC/大模型应用开发实习的本科生。

目标：不追求一次性学完所有 AI 理论，而是优先掌握能做出 Agent Demo、能写进简历、能在面试中讲清楚的核心能力。

---

## 1. 大模型 API 调用

你需要会：

- 调用大模型聊天接口。
- 设置 system/user/assistant messages。
- 理解 token、上下文长度、temperature、max tokens。
- 处理 API Key、环境变量、超时、重试、限流。
- 实现流式输出。
- 记录调用日志、耗时和成本。

最低实践目标：

- 用 Python 或 TypeScript 写一个 `/chat` 接口。
- 前端输入问题，后端调用大模型并返回回答。
- API Key 不写死在代码里。

---

## 2. Prompt 设计与结构化输出

你需要会：

- 设计清晰的 system prompt。
- 给模型明确角色、任务、约束和输出格式。
- 使用 few-shot 示例提升稳定性。
- 让模型在信息不足时明确拒答。
- 要求模型输出 JSON、Markdown 表格或固定字段。
- 对模型输出做解析和校验。

最低实践目标：

- 输入一段岗位 JD，输出结构化 JSON：

```json
{
  "skills": [],
  "responsibilities": [],
  "requirements": [],
  "difficulty": "low | medium | high"
}
```

面试要能讲清：

- 为什么普通自然语言输出不稳定。
- 为什么结构化输出便于后端处理。
- 如何减少模型乱编和格式错误。

---

## 3. Function Calling / Tool Calling 思想

你需要理解：

- 模型本身不会真正查数据库、搜网页、读文件，必须通过工具完成。
- Tool Calling 的核心是：让模型选择合适工具，并生成结构化参数。
- 工具执行结果再返回给模型，模型基于结果继续回答。

你需要会设计：

- 工具名称。
- 工具描述。
- 参数 schema。
- 返回结果格式。
- 错误处理。
- 调用日志。

常见工具例子：

- search_documents(query)
- search_jobs(keyword, city)
- parse_resume(file)
- query_database(sql)
- summarize_pdf(file_id)

最低实践目标：

- 写 3 个工具，让 Agent 能根据用户问题选择是否调用。
- 给工具参数做校验。
- 工具失败时返回明确错误信息。

面试要能讲清：

- Agent 和普通 Chatbot 的区别。
- 为什么工具调用需要权限控制。
- 如何避免 Agent 无限循环调用工具。

---

## 4. RAG 基础

RAG 是 Agent 实习最常见的项目能力。

完整流程：

```text
文档上传
-> 文档解析
-> 文本切分
-> 生成 embedding
-> 存入向量库
-> 用户提问
-> 向量检索
-> 拼接上下文
-> 大模型生成答案
-> 返回引用来源
```

你需要会：

- 读取 Markdown、TXT、PDF。
- 按段落、标题或固定长度切分文档。
- 调用 embedding 模型。
- 使用 Chroma、FAISS、Qdrant 或 Milvus 做向量检索。
- 根据检索结果生成问答。
- 返回答案引用来源。
- 分析 RAG 答错的原因。

最低实践目标：

- 做一个本地知识库问答 Demo。
- 支持上传文档。
- 支持基于文档提问。
- 回答时带引用片段或文件来源。

面试要能讲清：

- chunk size 太大或太小会有什么问题。
- 向量检索和关键词检索有什么区别。
- 为什么需要 rerank。
- RAG 如何减少幻觉。

---

## 5. 前后端基础：做一个可交互 Demo

Agent 项目不能只停留在脚本，最好能做成可交互 Demo。

后端需要会：

- FastAPI / Express / Next.js API。
- 接收 JSON 请求。
- 文件上传。
- 调用 LLM。
- 调用工具。
- 返回流式结果。
- 保存对话历史。

前端需要会：

- 输入框和聊天窗口。
- 文件上传。
- 展示模型回答。
- 展示引用来源。
- 展示工具调用过程。
- 展示加载状态和错误提示。

最低实践目标：

- 一个网页或 Streamlit/Gradio 页面。
- 用户可以上传文档并提问。
- 页面能显示回答、引用来源和执行状态。

推荐 Demo 页面结构：

```text
左侧：文档列表
中间：聊天窗口
右侧：引用来源 / 工具调用日志
```

---

## 6. Python 或 TypeScript 工程能力

Agent 开发至少要熟练一种主力语言。

### Python 路线

需要掌握：

- Python 基础语法。
- 虚拟环境和依赖管理。
- FastAPI。
- Pydantic。
- requests/httpx。
- SQLAlchemy 或 SQLModel。
- pytest。
- 日志 logging。

适合方向：

- RAG。
- Agent 工作流。
- 数据处理。
- AI 应用后端。

### TypeScript 路线

需要掌握：

- TypeScript 基础。
- Node.js。
- React / Next.js。
- API Route。
- zod 参数校验。
- fetch/axios。
- 前后端一体 Demo。

适合方向：

- AI Web 应用。
- Agent 产品原型。
- 全栈 Demo。

最低实践目标：

- 项目结构清晰。
- 配置和密钥分离。
- 核心逻辑模块化。
- 有基本异常处理。
- 有少量测试。

---

## 7. GitHub、README、部署和演示能力

很多实习筛选会看 GitHub 项目。项目能不能被别人跑起来很重要。

你需要会：

- 使用 Git 管理代码。
- 在 GitHub 创建仓库。
- 写清楚 README。
- 配置 `.gitignore`。
- 不上传 API Key。
- 使用环境变量。
- 提供启动命令。
- 提供截图或 Demo 视频。
- 使用 Docker 或云平台部署。

README 至少包含：

- 项目简介。
- 功能列表。
- 技术栈。
- 架构图或流程图。
- 本地运行方式。
- 环境变量说明。
- 示例截图。
- 已知问题。
- 后续计划。

简历中更好的写法：

```text
基于 FastAPI + LangGraph + Chroma 构建 Agentic RAG Demo，支持文档上传、向量检索、工具调用和带引用问答；提供可交互前端、Docker 启动方式和完整 README，并记录工具调用日志与模型响应耗时。
```

---

## 8. 学习优先级

建议顺序：

```text
大模型 API
-> Prompt 和结构化输出
-> FastAPI/前端 Demo
-> RAG
-> Tool Calling
-> Agent 工作流
-> README 和部署
```

暂时不用优先深挖：

- 从零训练大模型。
- 复杂强化学习。
- 多 Agent 论文细节。
- 大规模模型推理优化。

对于实习准备，先把一个能跑、能演示、能讲清楚的 Agent 项目做出来，比泛泛学习大量概念更重要。

