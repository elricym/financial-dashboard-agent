# Financial Dashboard Agent

基于 Eino Agent SDK (Go) 的金融看板生成系统。用户通过自然语言生成和编辑多面板金融看板。

## 设计文档

- [系统架构总览](docs/01-architecture.md)
- [Agent 设计](docs/02-agent-design.md)
- [数据源注册表](docs/03-data-source-registry.md)
- [DashboardState 状态模型](docs/04-dashboard-state.md)
- [沙箱执行环境](docs/05-sandbox.md)
- [前端渲染层](docs/06-frontend-rendering.md)
- [会话与编辑流程](docs/07-session-flow.md)
- [数据刷新与缓存](docs/08-data-refresh.md)
- [错误处理与降级](docs/09-error-handling.md)
- [安全模型](docs/10-security.md)

## 核心理念

**Code Mode** — 借鉴 [Cloudflare Code Mode MCP](https://blog.cloudflare.com/code-mode-mcp/) 思路，Agent 不从几十个 tool 中选择，而是通过代码在运行时发现和调用数据源。4 个 tool 覆盖所有场景，固定 ~1000 tokens 开销。

## 技术栈

| 层 | 技术 |
|---|---|
| Agent 框架 | [Eino SDK](https://github.com/cloudwego/eino) (Go) |
| 沙箱执行 | goja (JS-in-Go) |
| 状态存储 | Redis + PostgreSQL |
| 实时通信 | WebSocket |
| 前端图表 | TradingView Lightweight Charts + Apache ECharts |
| 前端布局 | gridstack.js |
| 前端框架 | React + TypeScript |
