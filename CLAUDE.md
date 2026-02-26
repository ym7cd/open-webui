# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Open WebUI is a self-hosted AI platform built with **SvelteKit** (frontend) and **FastAPI** (Python backend). It provides a web interface for LLM interactions with support for Ollama, OpenAI-compatible APIs, RAG, web search, image generation, and more.

## Development Commands

### Frontend (SvelteKit + TypeScript)
```bash
# Install dependencies
npm install

# Development server (with hot reload)
npm run dev          # Runs on http://localhost:5173
npm run dev:5050     # Alternative port

# Build for production
npm run build

# Type checking
npm run check        # Single run
npm run check:watch  # Watch mode

# Linting
npm run lint:frontend    # ESLint
npm run lint:types       # TypeScript checking

# Formatting
npm run format       # Prettier

# Tests
npm run test:frontend  # Vitest

# i18n (extract new translation keys)
npm run i18n:parse
```

### Backend (FastAPI + Python)
```bash
# From the backend/ directory

# Development server (with hot reload)
bash dev.sh
# Or manually:
export CORS_ALLOW_ORIGIN="http://localhost:5173;http://localhost:8080"
PORT=8080 uvicorn open_webui.main:app --port 8080 --host 0.0.0.0 --reload

# Linting
npm run lint:backend  # PyLint on backend/
npm run format:backend # Black formatter

# Production start
bash start.sh
```

### Docker (Local Deployment)
```bash
# 镜像已在当前目录构建完成，使用以下命令运行：
docker run -d \
  --name open-webui \
  -p 8081:8080 \
  -v open-webui-data:/app/backend/data \
  --add-host=host.docker.internal:host-gateway \
  --restart always \
  ghcr.io/open-webui/open-webui:main

# 或使用本地构建的镜像
docker run -d \
  --name open-webui \
  -p 8081:8080 \
  -v open-webui-data:/app/backend/data \
  --add-host=host.docker.internal:host-gateway \
  --restart always \
  open-webui:latest

# 访问地址: http://localhost:8081

# 容器管理
docker start open-webui      # 启动容器
docker stop open-webui       # 停止容器
docker rm -f open-webui      # 删除容器
docker logs -f open-webui    # 查看日志

# 数据卷管理
docker volume ls             # 列出所有卷
docker volume inspect open-webui-data  # 查看卷详情
```

## Architecture

### Frontend (`src/`)

- **SvelteKit** with Vite, using TypeScript
- **Routing**: File-based routing in `src/routes/`
  - `(app)/` - Main application routes (requires auth)
  - `auth/` - Authentication pages
  - `s/` - Shared/public routes
- **State Management**: Svelte stores in `src/lib/stores/index.ts`
  - Global state: `config`, `user`, `models`, `chats`, `settings`, etc.
- **Components**: `src/lib/components/`
  - `chat/` - Chat-related components (messages, input, settings)
  - `admin/` - Admin panel components
  - `common/` - Reusable UI components
  - `layout/` - Layout components (sidebar, header)
- **API Layer**: `src/lib/apis/index.ts` and subdirectories
  - Centralized API calls to backend
  - Streaming response handling for chat
- **i18n**: `src/lib/i18n/`
  - Translation files in `locales/{lang}/translation.json`
  - Add new languages: create directory, copy `en-US`, update `languages.json`
- **Utils**: `src/lib/utils/` - Helper functions

### Backend (`backend/open_webui/`)

- **FastAPI** application with Python 3.11+
- **Entry Point**: `main.py` - FastAPI app initialization
- **Routers**: `routers/` - API route modules
  - `auths.py` - Authentication endpoints
  - `chats.py` - Chat CRUD operations
  - `ollama.py` - Ollama integration
  - `openai.py` - OpenAI-compatible API
  - `models.py` - Model management
  - `retrieval.py` - RAG/document retrieval
  - `tools.py` - Tool/function calling
  - And more...
- **Models**: `models/` - Database models (peewee ORM)
- **Utils**: `utils/` - Backend utilities
  - `middleware.py` - Auth, rate limiting, CORS
  - `chat.py` - Chat message processing
  - `tools.py` - Tool execution logic
  - `misc.py` - Various helpers
- **Socket**: `socket/main.py` - WebSocket for real-time features
- **Retrieval**: `retrieval/` - RAG implementation
  - `vector/` - Vector database integrations
  - `web/` - Web search integration
  - `loaders/` - Document content extraction

### Key Integrations

- **LLM Backends**: Ollama (default), OpenAI-compatible APIs
- **Vector DBs**: ChromaDB, Qdrant, Milvus, Elasticsearch, etc.
- **Web Search**: 15+ providers (SearXNG, Brave, Tavily, etc.)
- **Image Gen**: DALL-E, ComfyUI, AUTOMATIC1111
- **Authentication**: JWT-based, OAuth, LDAP/AD, SSO
- **Storage**: SQLite (default), PostgreSQL, cloud (S3, GCS, Azure)

## Build Process

1. Frontend builds to `build/` directory (static files)
2. Dockerfile multi-stage build:
   - Stage 1: Build SvelteKit frontend (Node.js)
   - Stage 2: Python backend with embedded frontend
3. Backend serves both API (`/api/*`) and frontend (`/*`)

### Build Configuration Notice

**IMPORTANT: 构建镜像前必须确认模型配置**

- **模型使用方式**: 本项目仅使用第三方模型 API（如 OpenAI、Anthropic、通义千问等），无本地部署模型需求
- **构建选项**: 不启用 CUDA、不内置 Ollama
- **预下载模型**: 默认不预下载任何模型到镜像中
- **构建前确认**: 在执行 `docker build` 或类似构建命令前，必须按以下流程向用户询问：
  1. 首先询问是否需要预下载模型（精简版不预下载）
  2. 如果用户确认需要预下载 → 询问具体需要哪些模型
  3. 如果用户不需要 → 按默认配置构建（不包含模型文件）

## Configuration

- Environment variables: `.env` (see `.env.example`)
- Backend config: `backend/open_webui/config.py`
- Key env vars: `OLLAMA_BASE_URL`, `OPENAI_API_KEY`, `WEBUI_SECRET_KEY`, `CORS_ALLOW_ORIGIN`

### Local Docker Deployment Config
- **数据卷**: `open-webui-data` (持久化存储所有应用数据)
- **镜像**: 已构建在当前目录 `open-webui.tar`
- **端口映射**: 宿主机 `8081` → 容器内 `8080`
- **访问地址**: http://localhost:8081

**重启服务规则**：
```bash
# 重启服务默认使用最新镜像（当前目录下的 open-webui.tar）
docker load -i open-webui.tar && \
docker stop open-webui && docker rm open-webui && \
docker run -d \
  --name open-webui \
  -p 8081:8080 \
  -v open-webui-data:/app/backend/data \
  --add-host=host.docker.internal:host-gateway \
  --restart always \
  open-webui:latest
```

## Important Patterns

- **Streaming Responses**: Chat uses SSE/EventSource for streaming LLM responses
- **Auth Middleware**: JWT tokens stored in localStorage, sent via `Authorization: Bearer` header
- **i18n**: Use `$i18n.t('key')` in components; extract new keys with `npm run i18n:parse`
- **Database**: Peewee ORM with migration support via `peewee-migrate`
- **State Sync**: Svelte stores sync with backend on app init and user actions

## Testing

- **E2E**: Cypress (`cypress/`) - `npm run cy:open`
- **Frontend Unit**: Vitest - `npm run test:frontend`
- **Backend**: pytest (when installed)

## File Structure Notes

- Frontend static assets: `static/` (copied to build root)
- Backend static: `backend/open_webui/static/`
- i18n locales: `src/lib/i18n/locales/{lang}/translation.json`
- Python dependencies: `pyproject.toml` (managed with uv)
- Node dependencies: `package.json`

## Troubleshooting

### Web Search - ddgs 首次使用卡死

**症状**: 启用 DuckDuckGo (ddgs) 搜索后，页面卡死无响应

**原因**: `ddgs` 库首次使用时会初始化浏览器指纹（impersonate）数据，这个过程可能需要较长时间或超时

**解决方案**: 在容器中预先初始化 ddgs 库
```bash
docker exec open-webui python3 -c "
from ddgs import DDGS
with DDGS() as ddgs:
    list(ddgs.text('test', max_results=1))
print('ddgs 预热完成')
"
```

**预期输出**:
```
Impersonate 'firefox_xxx' does not exist, using 'random'
ddgs 预热完成
```

**注意**: 容器重启后可能需要重新执行预热命令
