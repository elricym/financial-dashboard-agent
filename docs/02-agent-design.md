# 02 - Agent 设计

## 设计原则

1. **最小 Tool 集** — 4 个 tool 覆盖全部场景，固定 token 开销 ~1000 tokens
2. **Code as Interface** — Agent 通过代码（而非预定义操作）表达意图，可处理任意用户需求
3. **渐进式发现** — Agent 不需要预先知道所有数据源，通过 search 按需发现
4. **状态感知** — 编辑场景下 Agent 了解当前看板状态，做精确修改
5. **Code not Data** — Agent 写的是数据获取代码，不是数据本身。Panel 存储 `code` 字段（JS 代码字符串），由渲染器在运行时执行以获取最新数据。

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
2. 用 `execute` 组装看板结构，为每个 panel 编写数据获取代码

### 编辑现有看板
1. 用 `get_state` 了解当前看板状态
2. 用 `mutate` 修改看板结构和 panel 代码

## 核心概念：Panel 存储 Code，不存储 Data
- 每个 panel 有一个 `code` 字段 — 一段 JS 代码字符串
- 后端在渲染时执行 panel.code 获取最新数据
- Agent 只需要编写正确的代码，不需要看到数据结果
- 刷新 = 重新执行 panel.code，无需 Agent 参与

## 代码编写规范
- 数据源通过 `sdk.<source>.<method>(params)` 调用
- 所有 SDK 调用都是异步的，使用 await
- 多个无依赖的调用使用 Promise.all 并行
- execute 返回完整的 DashboardState 对象，每个 panel 含 code 字段
- mutate 接收当前 state 作为参数，修改结构和 code 字符串，返回修改后的 state
- 使用 utils.* 中的工具函数进行常用计算
- panel.code 中可使用 sdk.* 和 utils.*

## DashboardState 结构
{
  title: string,
  layout: { columns: number, gap?: number },
  panels: [{
    id: string,           // 唯一标识，用于编辑时定位
    title: string,
    type: string,         // candlestick, line, area, bar, heatmap, pie, feed, table, metric_card...
    renderer: string,     // "lightweight-charts" | "echarts" | "custom"
    code: string,         // JS 代码字符串，渲染时执行以获取数据
    indicators?: [],      // 技术指标
    config?: {},          // 渲染配置
    dataSource?: { source, method, params }  // 可选元数据，描述数据来源
  }]
}

## 渲染器选择
- 金融时序图表（K线、价格走势）→ renderer: "lightweight-charts"
- 统计图表（热力图、饼图、雷达、散点）→ renderer: "echarts"
- 非图表内容（信息流、表格、指标卡片）→ renderer: "custom"

## 注意事项
- 始终为 panel 设置有意义的 id（英文，如 "gold-kline"、"btc-sentiment"）
- panel.code 应当是完整的、自包含的数据获取逻辑
- dataSource 作为可选元数据保留，方便 UI 展示来源信息
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

**用途：** 生成新的 DashboardState。Agent 编写结构代码，为每个 panel 指定 `code` 字段（数据获取代码字符串）。

```go
type ExecuteTool struct{}

type ExecuteInput struct {
    Code        string `json:"code" description:"JavaScript 代码，返回一个完整的 DashboardState 对象。每个 panel 的 code 字段是一段 JS 代码字符串，将在渲染时被执行以获取数据。Agent 不直接调用 SDK 获取数据，而是为每个 panel 编写数据获取代码。"`
    DashboardID string `json:"dashboard_id,omitempty" description:"可选。指定看板ID则替换，不指定则创建新看板。"`
}

type ExecuteOutput struct {
    DashboardID    string              `json:"dashboard_id"`
    State          DashboardState      `json:"state"`
    ValidationResult *ValidationResult `json:"validation,omitempty"` // 自动验证结果
    Error          string              `json:"error,omitempty"`
}

// ValidationResult 后端试运行 panel.code 的结果
type ValidationResult struct {
    Panels []PanelValidation `json:"panels"`
    AllOK  bool              `json:"allOk"`
}

type PanelValidation struct {
    PanelID  string `json:"panelId"`
    Status   string `json:"status"`   // "success" | "error"
    RowCount int    `json:"rowCount,omitempty"` // 数据行数
    Error    string `json:"error,omitempty"`
    Duration string `json:"duration"` // 执行耗时
}
```

**Eino Tool 注册：**
```go
executeTool := &eino.Tool{
    Name:        "execute",
    Description: "生成看板。写 JavaScript 代码返回 DashboardState，其中每个 panel 的 code 字段是数据获取代码字符串。后端会自动试运行所有 panel.code 并返回验证结果（成功/失败/行数）。如有失败，可用 mutate 修复。",
    InputSchema: ExecuteInput{},
    Handler:     executeHandler,
}
```

**执行逻辑：**
1. 创建 goja sandbox 实例
2. 执行 Agent 生成的 JS 代码（构建 DashboardState 结构）
3. 验证返回的 DashboardState 结构
4. **自动验证：** 试运行每个 panel.code，收集执行结果
5. 持久化到 State Store
6. 通过 WebSocket 推送到前端（后端执行 code 后注入 data）
7. 返回确认信息 + 验证结果摘要

### Tool 3: `get_state`

**用途：** 获取当前看板的状态信息。

```go
type GetStateTool struct{}

type GetStateInput struct {
    DashboardID string `json:"dashboard_id,omitempty" description:"看板ID。不指定则返回当前活跃看板。"`
    DetailLevel string `json:"detail_level" description:"'summary' 返回面板列表和关键配置（节省 token）；'full' 返回完整 state 含 code。" enum:"summary,full"`
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
        "indicators": ["MA(20)", "MA(60)"],
        "dataSource": { "source": "market", "method": "getKline" },
        "codePreview": "const data = await sdk.market.getKline({ symbol: 'XAUUSD', timeframe: '1D', limit: 120 }); return data;"
      },
      {
        "id": "gold-tweets",
        "title": "黄金舆情",
        "type": "feed",
        "renderer": "custom",
        "dataSource": { "source": "social", "method": "searchTweets" },
        "codePreview": "const tweets = await sdk.social.searchTweets({ query: 'gold', limit: 50 }); return { items: tweets };"
      }
    ]
  }
}
```

Summary 模式从完整 state 中提取关键字段，包含 code 预览（截断），节省 token。Agent 编辑时通常只需要 summary 就够了。

### Tool 4: `mutate`

**用途：** 修改当前看板状态。Agent 写代码直接操作 state 结构和 panel.code 字符串。

```go
type MutateTool struct{}

type MutateInput struct {
    Code        string `json:"code" description:"JavaScript 异步函数，接收当前 DashboardState 作为参数 state，返回修改后的 state。可以修改结构、增删 panel、修改 panel.code 字符串。不直接调用 sdk 获取数据。"`
    DashboardID string `json:"dashboard_id,omitempty" description:"看板ID。不指定则修改当前活跃看板。"`
}

type MutateOutput struct {
    DashboardID      string              `json:"dashboard_id"`
    Changes          []ChangeEntry       `json:"changes"`
    ValidationResult *ValidationResult   `json:"validation,omitempty"` // 自动验证结果
    Error            string              `json:"error,omitempty"`
}

type ChangeEntry struct {
    Type    string `json:"type"`    // "added_panel", "removed_panel", "modified_panel", "layout_changed"
    PanelID string `json:"panel_id,omitempty"`
    Detail  string `json:"detail"`
}
```

**执行逻辑：**
1. 从 State Store 读取当前 DashboardState
2. 创建 goja sandbox（注入 `utils`，不注入 `sdk` — Agent 不在 mutate 中直接调用数据）
3. 将当前 state 作为参数传入 Agent 代码
4. 执行代码，获取新 state
5. **Diff 检测**：对比新旧 state，生成 ChangeEntry 列表
6. 验证新 state 结构
7. **自动验证：** 对变更的 panel 试运行 panel.code，返回验证结果
8. 持久化新 state
9. 后端执行所有 panel.code 获取数据，推送到前端
10. 返回变更摘要 + 验证结果

**关键设计：Diff 而非全量推送。** mutate 执行后，系统对比新旧 state：
- 新增的 panel → 前端 append
- 删除的 panel → 前端 remove
- 修改的 panel → 前端局部刷新该 panel
- 布局变更 → 前端重新排列

这样避免每次编辑都全量重渲染。

## 自动验证机制

Agent 每次通过 execute 或 mutate 写入 state 后，后端自动试运行所有（或变更的）panel.code：

```go
func (h *Handler) validatePanelCodes(ctx context.Context, state *DashboardState, changedPanelIDs []string) *ValidationResult {
    result := &ValidationResult{AllOK: true}
    
    for _, panel := range state.Panels {
        // 如果指定了变更列表，只验证变更的 panel
        if len(changedPanelIDs) > 0 && !contains(changedPanelIDs, panel.ID) {
            continue
        }
        
        start := time.Now()
        data, err := h.sandbox.Execute(ctx, panel.Code) // 在沙箱中执行 panel.code
        duration := time.Since(start)
        
        pv := PanelValidation{
            PanelID:  panel.ID,
            Duration: duration.String(),
        }
        
        if err != nil {
            pv.Status = "error"
            pv.Error = err.Error()
            result.AllOK = false
        } else {
            pv.Status = "success"
            pv.RowCount = countRows(data) // 根据数据类型计算行数
        }
        
        result.Panels = append(result.Panels, pv)
    }
    
    return result
}
```

验证结果返回给 Agent，如果有 panel 执行失败，Agent 可以立即用 mutate 修复 code：

```
Agent execute → state 写入 → 后端试运行 panel.code → 返回验证结果
  ↓ 如果有失败
Agent mutate (修复失败的 panel.code) → 重新验证 → 成功 → 推送前端
```

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
- **execute/mutate 返回** — 裁剪为 summary + 验证结果摘要
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
