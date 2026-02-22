# 01 - 系统架构总览

## 系统全景

```
┌─────────────────────────────────────────────────────────────────┐
│                        客户端 (Browser)                          │
│                                                                 │
│  ┌──────────────┐  ┌──────────────────────────────────────────┐ │
│  │  Chat Panel   │  │         Dashboard Canvas                 │ │
│  │              │  │                                          │ │
│  │  用户输入     │  │  ┌─────────┐ ┌─────────┐ ┌─────────┐   │ │
│  │  Agent 回复   │  │  │ Panel A │ │ Panel B │ │ Panel C │   │ │
│  │  状态反馈     │  │  │ LW Chart│ │ ECharts │ │  Feed   │   │ │
│  │              │  │  └─────────┘ └─────────┘ └─────────┘   │ │
│  └──────┬───────┘  └──────────────────┬───────────────────────┘ │
│         │ HTTP/WS                      │ WS (state push)        │
└─────────┼──────────────────────────────┼────────────────────────┘
          │                              │
┌─────────▼──────────────────────────────▼────────────────────────┐
│                        API Gateway (Go)                          │
│                                                                  │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │ Chat API    │  │ Dashboard API│  │ WebSocket Hub          │  │
│  │ /chat       │  │ /dashboard/* │  │ /ws                    │  │
│  └──────┬──────┘  └──────┬───────┘  └────────────┬───────────┘  │
│         │                │                        │              │
│  ┌──────▼────────────────▼──────────────────┐     │              │
│  │          Session Manager                  │     │              │
│  │  - 会话上下文管理                          │     │              │
│  │  - 多看板状态关联                          │     │              │
│  │  - 消息历史                               │     │              │
│  └──────────────────┬───────────────────────┘     │              │
│                     │                              │              │
│  ┌──────────────────▼───────────────────────┐     │              │
│  │          Eino Agent Runtime               │     │              │
│  │                                           │     │              │
│  │  System Prompt + Tool Definitions         │     │              │
│  │  ┌────────┐ ┌─────────┐ ┌──────────┐     │     │              │
│  │  │ search │ │ execute │ │ get_state│     │     │              │
│  │  └────┬───┘ └────┬────┘ └────┬─────┘     │     │              │
│  │       │          │           │            │     │              │
│  │  ┌────┴──────────┴───────────┴──────┐     │     │              │
│  │  │         mutate                    │     │     │              │
│  │  └────────────────┬─────────────────┘     │     │              │
│  └───────────────────┼──────────────────────┘     │              │
│                      │                             │              │
│  ┌───────────────────▼──────────────────────┐     │              │
│  │        Sandbox Runtime (goja)             │     │              │
│  │                                           │     │              │
│  │  ┌─────────────┐  ┌──────────────────┐    │     │              │
│  │  │ SDK Proxy   │  │ State Accessor   │    │     │              │
│  │  │ Layer       │  │                  │    │     │              │
│  │  └──────┬──────┘  └────────┬─────────┘    │     │              │
│  │         │                  │              │     │              │
│  └─────────┼──────────────────┼──────────────┘     │              │
│            │                  │                     │              │
│  ┌─────────▼──────┐  ┌───────▼──────────────┐     │              │
│  │ Data Source    │  │ Dashboard State      │─────┘              │
│  │ Registry +    │  │ Store                │                    │
│  │ SDK Adapters  │  │ (Redis + PG)         │                    │
│  └───────┬───────┘  └──────────────────────┘                    │
│          │                                                       │
└──────────┼───────────────────────────────────────────────────────┘
           │
    ┌──────▼───────────────────────────────┐
    │        External Data Sources          │
    │                                       │
    │  行情API  财报API  舆情API  链上API    │
    │  新闻API  宏观API  持仓API  研报API    │
    │  ...（几十个）                         │
    └──────────────────────────────────────┘
```

## 核心模块职责

### API Gateway
- HTTP/WebSocket 入口
- 认证鉴权
- 请求路由
- 限流

### Session Manager
- 管理用户会话上下文（聊天历史、当前活跃看板）
- 一个会话可以关联多个 Dashboard
- 维护 Agent 对话的 message history

### Eino Agent Runtime
- LLM 调用（通过 Eino SDK）
- Tool 注册与调度
- System Prompt 管理
- Streaming 响应

### Sandbox Runtime
- Agent 生成的代码的安全执行环境
- 提供 SDK Proxy（数据源访问）和 State Accessor（看板状态读写）
- 资源隔离：内存、CPU、超时

### Data Source Registry
- 数据源元信息管理（spec）
- SDK Adapter 统一封装
- 请求代理与缓存

### Dashboard State Store
- DashboardState 的持久化
- 变更检测与推送（通过 WebSocket Hub）
- 版本历史（可选）

### WebSocket Hub
- 维护客户端连接
- Dashboard State 变更时推送增量更新
- Agent 思考/执行状态实时反馈

## 请求流转：生成看板

```
1. 用户在 Chat Panel 输入 "帮我做一个黄金分析看板"
2. Chat API → Session Manager → 构建 message history
3. Session Manager → Eino Agent Runtime → LLM 决策
4. Agent 调用 search tool → Sandbox 执行搜索代码 → 返回匹配数据源
5. Agent 调用 execute tool → Sandbox 执行数据拉取+组装代码
6. Sandbox → SDK Proxy → Data Source Adapters → External APIs
7. Sandbox 返回 DashboardState → State Store 持久化
8. State Store → WebSocket Hub → 推送到客户端
9. Dashboard Canvas 渲染看板
10. Agent 返回文字回复 "已为您生成黄金分析看板，包含..."
```

## 请求流转：编辑看板

```
1. 用户输入 "把K线换成周线，加个布林带"
2. Chat API → Session Manager → 构建 message history（含当前看板 context）
3. Agent 调用 get_state → 获取当前看板摘要
4. Agent 调用 mutate → Sandbox 执行修改代码
5. Sandbox 读取当前 state，执行 mutation，可能重新拉取数据
6. 返回新 DashboardState → State Store 更新
7. State Store 计算 diff → WebSocket Hub → 推送增量更新
8. Dashboard Canvas 局部刷新
```

## 部署架构

```
┌─────────────────────────────────────┐
│            Load Balancer             │
└──────────────┬──────────────────────┘
               │
    ┌──────────▼──────────┐
    │   API Gateway (Go)   │ × N (水平扩展)
    │   + Agent Runtime    │
    │   + Sandbox Pool     │
    └──────┬─────┬────────┘
           │     │
    ┌──────▼┐ ┌──▼──────┐
    │ Redis │ │PostgreSQL│
    │ (状态) │ │ (持久化) │
    └───────┘ └─────────┘
```

- Gateway 无状态，可水平扩展
- Redis 存热数据（活跃看板 state、缓存）
- PostgreSQL 存冷数据（历史看板、用户数据、审计日志）
- Sandbox 以池化方式管理，复用 goja runtime 实例
