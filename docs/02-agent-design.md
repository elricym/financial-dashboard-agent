# 02 - Agent 设计

## 设计原则

1. **最小 Tool 集** — 4 个 tool 覆盖全部场景，固定 token 开销 ~1000 tokens
2. **Code as Interface** — Agent 通过代码（而非预定义操作）表达意图，可处理任意用户需求
3. **渐进式发现** — Agent 不需要预先知道所有数据源，通过 search 按需发现
4. **状态感知** — 编辑场景下 Agent 了解当前看板状态，做精确修改

## System Prompt

```
你是一个金融看板生成助手。用户通过自然语言描述需求，你负责生成和编辑金融数据看板。

## 你的能力
- 生成包含多个面板(panel)的金融看板
- 支持的面板类型：K线图、折线图、面积图、柱状图、热力图、饼图、雷达图、表格、信息流、指标卡片等
- 可访问几十个金融数据源（行情、财报、舆情、宏观、链上等）
- 可编辑已有看板：修改、添加、删除面板，调整布局

## 工作流程

### 生成新看板
1. 用 `search` 找到需要的数据源和方法
2. 用 `execute` 拉取数据并组装看板

### 编辑现有看板
1. 用 `get_state` 了解当前看板状态
2. 用 `mutate` 修改看板（可以同时调用新数据）

## 代码编写规范
- 数据源通过 `sdk.<source>.<method>(params)` 调用
- 所有 SDK 调用都是异步的，使用 await
- 多个无依赖的调用使用 Promise.all 并行
- execute 返回完整的 DashboardState 对象
- mutate 接收当前 state 作为参数，返回修改后的 state
- 使用 utils.* 中的工具函数进行常用计算

## DashboardState 结构
{
  title: string,
  layout: { columns: number, gap?: number },
  panels: [{
    id: string,           // 唯一标识，用于编辑时定位
    title: string,
    type: string,         // candlestick, line, area, bar, heatmap, pie, feed, table, metric_card...
    renderer: string,     // "lightweight-charts" | "echarts" | "custom"
    data: any,
    indicators?: [],      // 技术指标
    config?: {},          // 渲染配置
    dataSource?: { source, method, params }  // 记录来源，用于刷新
  }]
}

## 渲染器选择
- 金融时序图表（K线、价格走势）→ renderer: "lightweight-charts"
- 统计图表（热力图、饼图、雷达、散点）→ renderer: "echarts"
- 非图表内容（信息流、表格、指标卡片）→ renderer: "custom"

## 注意事项
- 始终为 panel 设置有意义的 id（英文，如 "gold-kline"、"btc-sentiment"）
- 始终记录 dataSource，以便后续刷新
- 编辑时尽量保留用户已有的 panel，只修改需要变更的部分
- 如果用户需求不明确，先回复确认再生成
```

## Tool 定义

### Tool 1: `search`

**用途：** 在数据源注册表中搜索可用的 SDK 和方法。

```go
type SearchTool struct{}

type SearchInput struct {
    Code string `json:"code" description:"JavaScript 代码，在数据源 spec 上执行搜索。spec.sources 是所有数据源的数组，每个 source 有 name, displayName, description, tags[], methods[] 字段。每个 method 有 name, description, params[], returns 字段。返回匹配到的数据源和方法信息。"`
}

type SearchOutput struct {
    Results []DataSourceMatch `json:"results"`
    Error   string            `json:"error,omitempty"`
}

type DataSourceMatch struct {
    Source      string       `json:"source"`
    Method      string       `json:"method"`
    Description string       `json:"description"`
    Params      []ParamSpec  `json:"params"`
    Returns     ReturnSpec   `json:"returns"`
}
```

**Eino Tool 注册：**
```go
searchTool := &eino.Tool{
    Name:        "search",
    Description: "搜索可用的数据源。写 JavaScript 代码在 spec 中搜索，发现需要的 SDK 和方法。spec.sources[] 包含所有数据源的元信息。",
    InputSchema: SearchInput{},
    Handler:     searchHandler,
}
```

**执行逻辑：**
1. 创建 goja sandbox 实例
2. 注入 `spec` 对象（数据源注册表）
3. 执行 Agent 生成的 JS 代码
4. 返回搜索结果

### Tool 2: `execute`

**用途：** 执行数据查询并生成新的 DashboardState。

```go
type ExecuteTool struct{}

type ExecuteInput struct {
    Code        string `json:"code" description:"JavaScript 异步函数体。通过 sdk.<source>.<method>(params) 调用数据源，返回一个完整的 DashboardState 对象。支持 await 和 Promise.all。可使用 utils.* 工具函数。"`
    DashboardID string `json:"dashboard_id,omitempty" description:"可选。指定看板ID则替换，不指定则创建新看板。"`
}

type ExecuteOutput struct {
    DashboardID string         `json:"dashboard_id"`
    State       DashboardState `json:"state"`
    Error       string         `json:"error,omitempty"`
}
```

**Eino Tool 注册：**
```go
executeTool := &eino.Tool{
    Name:        "execute",
    Description: "执行代码生成看板。写 JavaScript 代码调用数据源 SDK，组装并返回 DashboardState。代码中通过 sdk.market.getKline() 等方式调用数据，通过 utils.* 使用工具函数。返回的对象会自动保存并推送到前端渲染。",
    InputSchema: ExecuteInput{},
    Handler:     executeHandler,
}
```

**执行逻辑：**
1. 创建 goja sandbox 实例
2. 注入 `sdk`（数据源代理）、`utils`（工具函数）
3. 执行 Agent 生成的 JS 代码
4. 验证返回的 DashboardState 结构
5. 持久化到 State Store
6. 通过 WebSocket 推送到前端
7. 返回确认信息

### Tool 3: `get_state`

**用途：** 获取当前看板的状态信息。

```go
type GetStateTool struct{}

type GetStateInput struct {
    DashboardID string `json:"dashboard_id,omitempty" description:"看板ID。不指定则返回当前活跃看板。"`
    DetailLevel string `json:"detail_level" description:"'summary' 返回面板列表和关键配置（节省 token）；'full' 返回完整 state 含数据。" enum:"summary,full"`
}

type GetStateOutput struct {
    DashboardID string      `json:"dashboard_id"`
    State       interface{} `json:"state"` // summary 或 full
    Error       string      `json:"error,omitempty"`
}
```

**Summary 模式输出示例：**
```json
{
  "dashboard_id": "db_abc123",
  "state": {
    "title": "黄金分析看板",
    "layout": { "columns": 2 },
    "panels": [
      {
        "id": "gold-kline",
        "title": "黄金日线",
        "type": "candlestick",
        "renderer": "lightweight-charts",
        "timeframe": "1D",
        "symbol": "XAUUSD",
        "indicators": ["MA(20)", "MA(60)"],
        "dataSource": { "source": "market", "method": "getKline" }
      },
      {
        "id": "gold-tweets",
        "title": "黄金舆情",
        "type": "feed",
        "renderer": "custom",
        "itemCount": 50,
        "dataSource": { "source": "social", "method": "searchTweets" }
      }
    ]
  }
}
```

Summary 模式从完整 state 中提取关键字段，省略大块 data，节省 token。Agent 编辑时通常只需要 summary 就够了。

### Tool 4: `mutate`

**用途：** 修改当前看板状态。Agent 写代码直接操作 state，可以做任意修改。

```go
type MutateTool struct{}

type MutateInput struct {
    Code        string `json:"code" description:"JavaScript 异步函数，接收当前 DashboardState 作为参数 state，返回修改后的 state。可以修改任何字段、增删 panel、调用 sdk 获取新数据。支持 await 和 Promise.all。"`
    DashboardID string `json:"dashboard_id,omitempty" description:"看板ID。不指定则修改当前活跃看板。"`
}

type MutateOutput struct {
    DashboardID string         `json:"dashboard_id"`
    Changes     []ChangeEntry  `json:"changes"` // 描述做了什么变更
    Error       string         `json:"error,omitempty"`
}

type ChangeEntry struct {
    Type    string `json:"type"`    // "added_panel", "removed_panel", "modified_panel", "layout_changed"
    PanelID string `json:"panel_id,omitempty"`
    Detail  string `json:"detail"`
}
```

**执行逻辑：**
1. 从 State Store 读取当前 DashboardState
2. 创建 goja sandbox，注入 `sdk`、`utils`
3. 将当前 state 作为参数传入 Agent 代码
4. 执行代码，获取新 state
5. **Diff 检测**：对比新旧 state，生成 ChangeEntry 列表
6. 验证新 state 结构
7. 持久化新 state
8. 计算增量更新，通过 WebSocket 推送
9. 返回变更摘要

**关键设计：Diff 而非全量推送。** mutate 执行后，系统对比新旧 state：
- 新增的 panel → 前端 append
- 删除的 panel → 前端 remove
- 修改的 panel → 前端局部刷新该 panel
- 布局变更 → 前端重新排列

这样避免每次编辑都全量重渲染。

## Agent 运行时配置

```go
agentConfig := &eino.AgentConfig{
    Model:       "claude-sonnet-4-20250514", // 或其他适合的模型
    Temperature: 0.1,                     // 低温度，代码生成需要确定性
    MaxTokens:   4096,
    Tools:       []eino.Tool{searchTool, executeTool, getStateTool, mutateTool},
    SystemPrompt: systemPrompt,           // 上面定义的 system prompt
}
```

## 上下文管理

### Message History 策略

```
[System Prompt]                          // 固定，~800 tokens
[Tool Definitions × 4]                   // 固定，~1000 tokens
[Dashboard Context]                      // 动态，当前看板 summary，~200-500 tokens
[Conversation History]                   // 动态，最近 N 轮对话
[User Message]                           // 当前用户输入
```

**Dashboard Context** 在每轮对话自动注入：
```
当前活跃看板: "黄金分析看板" (db_abc123)
面板: gold-kline (黄金日线 K线), gold-sentiment (黄金舆情 信息流)
```

这让 Agent 无需每次调用 `get_state` 就知道当前看板概况。只有需要详细信息时才调用。

### History 裁剪

对话历史中，tool call 的完整代码和返回数据会快速消耗 context。策略：
- **search 结果** — 保留完整（通常较短）
- **execute/mutate 代码** — 保留完整（Agent 可能参考之前的代码模式）
- **execute/mutate 返回的 state** — 裁剪为 summary，移除 data 字段
- 超过 N 轮（如 20 轮）的历史 → 保留 system prompt + 最近 N 轮

## 多看板管理

用户可以创建多个看板，通过对话切换：

```go
type SessionContext struct {
    UserID           string
    ActiveDashboardID string            // 当前活跃看板
    Dashboards       []DashboardRef     // 所有看板列表
    MessageHistory   []eino.Message
}

type DashboardRef struct {
    ID    string
    Title string
}
```

Agent 通过对话理解用户意图来切换看板：
- "新建一个加密货币看板" → 创建新 Dashboard，设为活跃
- "回到黄金看板" → 切换 ActiveDashboardID
- "删掉这个看板" → 删除当前活跃看板

不需要额外的 tool，这些操作通过 execute（新建）和 mutate（修改/删除）完成。
