# CLAUDE.md

本文件为 Claude Code（claude.ai/code）在此仓库中工作时提供指导。

## 常用命令

**安装依赖（始终使用 `uv`，禁止使用 `pip`）：**
```bash
uv sync          # 从 uv.lock 安装依赖
uv add <pkg>     # 添加新依赖
uv remove <pkg>  # 移除依赖
```

**运行应用（始终使用 `uv`，禁止使用 `pip` 或直接运行 `python`）：**
```bash
./run.sh
# 或手动执行：
cd backend && uv run uvicorn app:app --reload --port 8000
```

**访问地址：**
- Web 界面：http://localhost:8000
- API 文档：http://localhost:8000/docs

**环境配置：** 将 `.env.example` 复制为 `.env`，并填入你的 `ANTHROPIC_API_KEY`。

## 架构说明

这是一个基于课程材料的 RAG（检索增强生成）聊天机器人。后端使用 FastAPI + ChromaDB + Anthropic Claude；前端使用原生 HTML/CSS/JS。

### 数据流

```
用户查询 → POST /api/query → RAGSystem.query()
  → Claude API（函数调用）→ CourseSearchTool.search()
  → ChromaDB 向量检索 → Claude 综合生成回答
  → 答案 + 来源 → 前端
```

### 后端核心模块（`backend/`）

- **`rag_system.py`** — 核心编排器；协调文档摄入、检索和 AI 生成。所有查询的入口。
- **`ai_generator.py`** — 封装 Anthropic Claude 的工具调用（函数调用）。Claude 调用 `CourseSearchTool` 检索相关文本块，而非直接在提示词中接收。模型：`claude-sonnet-4-20250514`，temp=0，max_tokens=800。
- **`vector_store.py`** — ChromaDB，含两个集合：`course_catalog`（课程标题/元数据）和 `course_content`（分块文本）。支持模糊课程名称匹配和课时过滤。
- **`document_processor.py`** — 解析课程文档，期望格式如下：
  ```
  Course Title: ...
  Course Link: ...
  Course Instructor: ...
  Lesson N: Title
  [内容]
  ```
  使用句子感知分割，每块 800 字符，重叠 100 字符。
- **`search_tools.py`** — Claude 函数调用的工具接口；封装向量检索，支持课程/课时过滤。追踪来源以供 UI 展示。
- **`session_manager.py`** — 内存中的会话对话历史（默认最多保留 2 轮对话）。
- **`config.py`** — 所有可调常量（块大小、模型名称、最大结果数等）。

### API 接口

- `POST /api/query` — `{query, session_id?}` → `{answer, sources, session_id}`
- `GET /api/courses` — 返回课程数量和课程名称列表
- `GET /` — 提供前端静态文件服务

### 前端（`frontend/`）

原生 JS，无需构建步骤。使用 `marked.js` 渲染 Markdown。通过相对 URL 与后端通信，可在代理后正常工作。

### 存储

ChromaDB 持久化到 `backend/chroma_db/`。删除该目录可重置向量数据库。示例课程文档存放于 `docs/`，启动时自动加载。
