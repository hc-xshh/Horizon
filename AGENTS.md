# Horizon

> AI 驱动的情报聚合系统。Fetch → Score → Filter → Enrich → Summarize，生成中英文每日简报。
> 
> **本项目为 clone**（上游：Thysrael/Horizon）。通过 Hermes MCP 集成使用，daily-hub 和定时任务的核心引擎。

## Build & Run

```bash
# 通过 MCP wrapper 运行（Hermes 集成方式）
./horizon-mcp-wrapper.sh

# 直接跑
uv run horizon

# 测试
uv run pytest tests/ -v
```

**注意**：git 操作需要先修复 ownership（当前 root clone）：
```bash
git config --global --add safe.directory /home/haha/workspace/Horizon
```

## Architecture

```
src/
├── main.py           # CLI 入口
├── models.py         # Pydantic 数据模型（Item, Stage, Run）
├── orchestrator.py   # 编排器：fetch → score → filter → enrich → summarize
└── search.py         # 搜索模块
tests/                # pytest（19 个测试文件）
docs/                 # 配置指南、评分算法、爬虫文档
scripts/
└── check_mcp.py      # MCP 状态检查
```

## Pipeline (5 阶段)

```
Fetch   → 从多个信息源采集原始内容（RSS, Twitter, Reddit, arXiv, GitHub 等）
Score   → LLM 评分（多模型支持：Claude, GPT, Gemini, DeepSeek, Doubao, MiniMax）
Filter  → 按阈值 + 话题去重筛选
Enrich  → 丰富条目信息
Summary → 生成结构化中英文简报
```

## MCP 集成

Hermes 通过 MCP 服务器 `horizon` 调用，暴露的工具：
- `hz_run_pipeline` — 一键跑全流程
- `hz_fetch_items` / `hz_score_items` / `hz_filter_items` / `hz_enrich_items` — 分阶段调用
- `hz_generate_summary` — 生成简报
- `hz_get_run_stage` / `hz_get_run_meta` — 查看运行状态
- `hz_validate_config` — 验证配置

## 与 daily-hub 的关系

```
Horizon MCP (采集+评分+简报)
    ↓ JSON 输出
daily-hub/scripts/parse_horizon.py (解析)
    ↓ 生成 public/data/*.json
daily-hub Python 加固脚本 (harden/audit)
    ↓
Vite 构建 → Cloudflare Pages
```

## Tech Stack

- Python 3.11+, uv 包管理
- httpx, feedparser (RSS), beautifulsoup4 (HTML)
- anthropic / openai / google-genai (多 LLM 后端)
- tenacity (重试), rich (CLI UI)
- MCP 协议 (Model Context Protocol)

## Pitfalls

- **ownership 问题**：clone 时 root 用户，git 操作需 `safe.directory` 配置
- **API 密钥**：需要各 LLM 提供商的 API key（通过环境变量或 `.env`）
- **上游更新**：这是 clone，上游有更新时需手动 `git pull`（注意不要覆盖本地配置）
- **MCP 超时**：大项目索引时 MCP 可能超时，用命令行方式调用替代
- **Twitter cookie**：需要有效的 Twitter 认证才能抓取，见 `docs/twitter-cookies.md`

## Cron

- 每日 05:00 CST → Horizon 全流程 → daily-hub 消费
- MCP wrapper 路径：`/home/haha/workspace/Horizon/horizon-mcp-wrapper.sh`
