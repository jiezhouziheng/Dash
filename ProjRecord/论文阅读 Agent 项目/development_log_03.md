# Paper-Reading-Agent 开发日志 03：接入真实 LLM 与测试隔离

日期：2026-05-30

## 1. 本阶段目标

本阶段目标是把 Paper-Reading-Agent 从“有完整工程链路但 LLM 仍是占位”的状态，推进到“能够真实调用模型生成论文分析”的状态。

上一阶段已经完成了 Markdown 输入、章节解析、文本分块、关键词检索问答、阅读笔记渲染和 pytest 测试体系。也就是说，系统已经有了论文阅读 Agent 的本地数据流：

```text
title / abstract / markdown_body
  -> PaperService
  -> paper_id
  -> paper.md
  -> sections
  -> chunks
  -> analysis.json
  -> reading_note.md
  -> ask 接口基于 chunks 返回 citations
```

本阶段重点不是重写这条链路，而是在已有边界上替换最关键的智能能力：

```text
LLMClient 占位输出
  -> OpenAI SDK 真实模型调用
  -> Pydantic 结构化解析
  -> 失败时回退占位输出
```

## 2. 本阶段关键成果

### 2.1 引入 OpenAI SDK 和 `.env` 配置

新增依赖：

```text
openai
```

项目现在通过 `.env` 管理模型调用配置：

```env
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-5.5
# OPENAI_BASE_URL=https://your-compatible-endpoint/v1
```

设计原因：

- API Key 不能写死在代码里。
- 模型名不应该硬编码，方便后续切换模型。
- `OPENAI_BASE_URL` 预留给兼容 OpenAI 协议的服务。
- `.env` 已被 `.gitignore` 忽略，避免密钥进入仓库。

这里也踩到了一个重要问题：如果使用 OpenAI 官方 API，不能在 `.env` 里保留空值形式的 `OPENAI_BASE_URL=`。OpenAI SDK 会读取这个环境变量，并把空字符串当成 base url，最终报：

```text
Request URL is missing an 'http://' or 'https://' protocol.
```

解决方式是：

```env
# OPENAI_BASE_URL=
```

如果使用兼容服务，则必须填写完整 URL：

```env
OPENAI_BASE_URL=https://example.com/v1
```

### 2.2 让 `LLMClient` 成为真实模型调用边界

本阶段继续坚持“模型调用不能散落到服务层或 API 层”的原则，只改 `backend/app/llm/client.py` 作为模型边界。

当前 `LLMClient` 负责：

- 读取 `.env`。
- 校验 API Key 是否有效。
- 创建 OpenAI SDK client。
- 构造论文分析 prompt。
- 限制输入正文长度。
- 使用 OpenAI `responses.parse()` 获取结构化输出。
- 将模型输出转换为项目内部 JSON 结构。
- 调用失败时回退占位分析。

这样做的好处是，`PaperService` 不需要知道 OpenAI 的细节，也不需要处理网络错误、认证错误或模型输出解析问题。服务层只关心：

```python
llm_analysis = self.llm_client.analyze_paper(...)
summary = llm_analysis["summary"]
learning_plan = llm_analysis["learning_plan"]
glossary = llm_analysis["glossary"]
```

### 2.3 从三次模型调用风险改为一次完整分析调用

旧设计里有三个方法：

```python
summarize_paper()
create_learning_plan()
extract_glossary()
```

如果后续每个方法都真实调用一次 LLM，那么一次论文分析就会变成三次 API 请求。这个设计有几个问题：

- 成本更高。
- 速度更慢。
- 三次输出可能互相不一致。
- 错误处理会分散。

因此本阶段新增统一入口：

```python
analyze_paper(title, abstract, markdown_body)
```

一次返回完整结构：

```python
{
    "summary": {...},
    "learning_plan": [...],
    "glossary": [...]
}
```

旧的三个方法暂时保留为兼容接口，但 `PaperService.analyze()` 已经改为只调用一次 `analyze_paper()`。

这个改动体现了一个重要设计思路：真实模型调用应该围绕完整任务设计，而不是把一个分析任务拆成多个互相独立的模型请求。

### 2.4 使用 Pydantic 约束 LLM 输出

新增结构：

```python
class LLMAnalysisResult(BaseModel):
    summary: Summary
    learning_plan: list[LearningTask]
    glossary: list[GlossaryItem]
```

这里复用了已有 schema：

- `Summary`
- `LearningTask`
- `GlossaryItem`

模型输出不是直接进入系统，而是必须经过：

```text
LLM raw output
  -> LLMAnalysisResult
  -> Summary / LearningTask / GlossaryItem
  -> PaperAnalysisResult
  -> analysis.json / reading_note.md / frontend
```

这样做的意义是：

- 前端可以依赖稳定字段。
- JSON 落盘结构稳定。
- 后续测试和重构更容易。
- LLM 输出不稳定的问题被限制在模型边界内。

这也是本项目简历表达里最值得强调的点之一：不是简单“调用了大模型 API”，而是用 schema-first 的方式把模型输出纳入工程系统。

### 2.5 Prompt 设计：约束模型只基于论文输入

本阶段新增了论文分析 prompt，核心要求包括：

- 只基于输入内容分析，不编造论文中没有的信息。
- 信息不足时明确写“原文信息不足”。
- `summary` 覆盖研究问题、方法、贡献、局限。
- `learning_plan` 给出具体可执行任务。
- `glossary` 用适合本科生理解的语言解释术语。
- 输出必须符合结构，不输出额外说明。

这和论文阅读 Agent 的目标一致：用户需要的是可信、可复习、可继续学习的阅读辅助，而不是泛泛聊天。

同时加入了正文截断：

```text
默认最多传入 12000 字符
```

当前只是最小实现，后续更合理的方案是：

```text
Markdown / PDF
  -> sections
  -> chunks
  -> 选择关键 chunks
  -> LLM 分析
```

### 2.6 增加模型调用失败回退

真实模型调用可能失败：

- API Key 无效。
- 模型名无权限。
- 网络连接失败。
- 服务限流。
- 输出无法解析成 schema。

因此 `LLMClient` 在模型调用失败时会：

```text
打印错误信息
回退到占位分析
继续完成解析、分块、落盘和阅读笔记生成
```

这个策略的设计原因是：对当前本地 MVP 来说，论文解析和产物落盘是基础能力，不能因为外部服务短暂失败就让整个流程中断。

### 2.7 测试隔离：自动化测试不调用真实 LLM

接入真实 LLM 后，原有测试一开始出现了一个风险：只要本地 `.env` 有 API Key，`pytest` 就可能真的调用模型。

这样会带来问题：

- 测试变慢。
- 测试依赖网络。
- 测试可能产生费用。
- CI 或他人环境缺少 Key 时测试不稳定。

解决方案是在 `backend/tests/conftest.py` 中新增：

```python
FakeLLMClient
```

所有服务层、API、问答和 renderer 测试都通过依赖注入显式使用 fake LLM：

```python
PaperService(
    file_store=FileStore(root_dir=tmp_path),
    llm_client=fake_llm_client,
)
```

这体现了依赖注入的价值：

```text
生产环境：PaperService() -> LLMClient() -> 真实模型或占位回退
测试环境：PaperService(llm_client=FakeLLMClient()) -> 稳定本地假数据
```

当前后端测试保持：

```text
17 passed
```

## 3. 遇到的问题与解决思路

### 3.1 `.env` 文件命名错误导致密钥可能不被忽略

最初本地出现了：

```text
about_openai.env
```

而 `.gitignore` 只忽略：

```text
.env
```

如果 API Key 写在 `about_openai.env` 中，就有被误提交的风险。解决方式是把文件改名为 `.env`，并确认 `.gitignore` 中包含 `.env`。

经验：

密钥文件不只是“能被代码读到”就行，还必须确认不会进入 Git。

### 3.2 中文占位 API Key 被当成真实 Key

`.env` 中曾出现类似：

```env
OPENAI_API_KEY=你的 API Key
```

代码最初只判断 `OPENAI_API_KEY` 是否为空。只要非空，就创建 OpenAI client。结果 SDK 构造 HTTP header 时，中文字符导致：

```text
UnicodeEncodeError
```

解决方式是增加 Key 有效性判断：

- 空值返回 False。
- 常见占位值返回 False。
- 不能编码为 ASCII 的 Key 返回 False。

这个问题说明：配置不是“有值就一定有效”。本地开发阶段经常会有占位配置，代码需要识别这些非真实配置。

### 3.3 忘记初始化 `self.client = None`

增加 Key 校验后，又出现了：

```text
AttributeError: 'LLMClient' object has no attribute 'client'
```

原因是只有 Key 有效时才设置：

```python
self.client = OpenAI(...)
```

Key 无效时 `self.client` 根本不存在。解决方式是在判断前统一初始化：

```python
self.client = None
```

经验：

对象属性应该在构造阶段形成稳定形态，不应该只在某些分支里存在。否则后续代码需要到处判断属性是否存在。

### 3.4 `OPENAI_BASE_URL=` 空值导致 SDK 请求地址错误

`.env` 中写了：

```env
OPENAI_BASE_URL=
```

代码里虽然把空字符串转成了 `None`，但 OpenAI SDK 自己也会读取环境变量。它读到空字符串后，最终报：

```text
Request URL is missing an 'http://' or 'https://' protocol.
```

解决方式：

- 使用 OpenAI 官方 API 时，注释掉或删除 `OPENAI_BASE_URL`。
- 使用兼容服务时，填写完整 URL，例如 `https://example.com/v1`。

经验：

第三方 SDK 也会读取环境变量。项目代码处理了配置，不代表 SDK 内部不会再读一次。

### 3.5 API Key 无效导致 401

修复 base url 后，OpenAI 返回：

```text
401 invalid_api_key
```

这说明网络和 endpoint 已经通了，问题转移到认证。解决方式是重新在 OpenAI Platform 创建有效 API Key，并确认 `.env` 中没有引号、空格、注释或旧 Key。

经验：

调 API 时要分层判断：

```text
URL 错误 -> 配置问题
401 -> Key 或权限问题
model not found -> 模型名或账号权限问题
connection error -> 网络或沙箱问题
schema parse error -> 输出结构或 prompt 问题
```

### 3.6 沙箱网络导致本地调用失败

普通命令中真实 LLM 调用曾出现：

```text
Connection error
```

在授权联网后，相同代码成功返回结构化结果。说明代码本身可用，问题在运行环境的网络权限。

经验：

在 Codex/受限环境中，涉及外部 API 的验证需要区分“代码问题”和“环境权限问题”。自动化测试不应依赖外网，真实 API 验证应作为手动联调步骤。

### 3.7 HTTP 服务写入 `data/papers` 时遇到 Windows 权限问题

通过 uvicorn 访问 `/api/papers/analyze` 时，服务进程曾报：

```text
PermissionError: [WinError 5]
```

但同一个 Python 解释器直接调用 `PaperService().analyze()` 可以成功写入 `data/papers` 和 `data/outputs`。这说明核心业务代码和文件落盘逻辑是通的，问题更接近 Windows/沙箱下 uvicorn 进程的文件权限差异。

当前验证方式：

- 服务层直接调用真实 LLM 并落盘成功。
- `analysis.json` 和 `reading_note.md` 均生成正常中文内容。
- 自动化测试仍保持 `17 passed`。

后续如果继续做前端联调，需要用有文件写入权限的方式启动 uvicorn，或将运行期数据目录配置到明确可写的位置。

### 3.8 PowerShell 中文输出乱码

真实 LLM 返回中文后，`python -c ... print(r)` 在终端里显示为乱码。但用：

```powershell
Get-Content -Encoding UTF8
```

读取 `analysis.json` 和 `reading_note.md` 时，中文内容正常。

经验：

Windows 终端显示乱码不等于文件编码错误。对中文 Markdown/JSON 产物，应使用 UTF-8 读回验证。

## 4. 当前架构快照

```text
frontend/index.html + main.js
  -> POST /api/papers/analyze
      -> PaperAnalyzeRequest
      -> PaperService.analyze()
          -> FileStore.prepare_paper_workspace()
          -> save paper.md
          -> parse_markdown_sections()
          -> build_text_chunks()
          -> LLMClient.analyze_paper()
              -> load .env
              -> validate api key
              -> build prompt
              -> responses.parse(text_format=LLMAnalysisResult)
              -> fallback placeholder on failure
          -> PaperAnalysisResult
          -> save analysis.json
          -> save chunks.json
          -> render_reading_note()
          -> save reading_note.md
      -> response_model=PaperAnalysisResult

  -> POST /api/papers/{paper_id}/ask
      -> PaperQuestionRequest
      -> PaperService.ask()
          -> FileStore.load_json(chunks.json)
          -> KeywordRetriever.retrieve()
          -> PaperQuestionAnswer
      -> response_model=PaperQuestionAnswer
```

## 5. 当前能力与限制

### 已具备能力

- Markdown 论文输入。
- Markdown 原文保存。
- 章节解析。
- 文本分块。
- OpenAI SDK 真实 LLM 分析。
- Pydantic 结构化输出校验。
- 分析 JSON 落盘。
- 阅读笔记 Markdown 渲染。
- 基于 chunks 的关键词问答。
- citations 返回。
- pytest 自动化测试。
- 测试 fake LLM 隔离。

### 仍然存在的限制

- 尚未支持 PDF 解析。
- 问答仍是关键词检索和模板回答，不是 LLM 生成式回答。
- chunks 的章节归属仍较粗略。
- 尚未保存问答历史。
- 尚未引入 SQLite 管理论文元数据。
- 尚未引入向量检索。
- 前端仍是原生静态页面，适合当前 MVP 阶段。

## 6. 简历表达建议

可以把当前项目描述为：

> 设计并实现一个本地优先的论文阅读 Agent MVP，基于 FastAPI 和 Pydantic 构建结构化 API，支持 Markdown 论文解析、章节提取、文本分块、本地文件持久化、OpenAI SDK 结构化模型调用和基于关键词检索的可追溯问答；通过 `LLMAnalysisResult` 对模型输出进行 schema 校验，并使用依赖注入和 `FakeLLMClient` 隔离测试环境，保证核心流程在无网络和无 API Key 情况下仍可稳定测试。

可突出技术点：

- FastAPI API 设计。
- Pydantic schema-first 数据建模。
- OpenAI SDK 结构化输出解析。
- Prompt 构造和模型调用边界封装。
- LLM 调用失败回退策略。
- 本地文件系统持久化。
- Markdown 解析和文本分块。
- RAG 前置链路：chunks、retriever、citations。
- pytest 测试和依赖注入。
- Windows 本地环境问题排查。

可突出工程能力：

- 没有把 OpenAI 调用直接写进 API 或服务层，而是封装在 `LLMClient`。
- 没有让 LLM 输出直接污染系统，而是经过 Pydantic schema 校验。
- 没有让测试依赖真实 API，而是用 fake client 隔离外部服务。
- 遇到配置、认证、网络、权限、编码问题时，逐层定位并保留可复现解决方案。

## 7. 下一阶段计划

建议下一阶段继续按以下顺序推进：

1. 将 `/api/papers/{paper_id}/ask` 升级为“关键词检索 citations -> LLM 基于引用片段生成答案”。
2. 保存问答历史到 `qa.json` 或后续 SQLite。
3. 优化 chunking，使每个 chunk 精确归属对应 Markdown section。
4. 增加 PDF parser，把 PDF 解析结果转换为统一 Markdown/sections/chunks。
5. 引入 SQLite，保存论文元数据、分析状态和历史记录。
6. 引入 embedding 和向量检索，替换或增强 `KeywordRetriever`。
7. 扩展前端为更完整的阅读工作台。
