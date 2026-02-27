# 04 - DashboardState 状态模型

## 概述

DashboardState 是整个系统的核心数据结构，是前端渲染的 single source of truth。Agent 生成和编辑看板的本质就是构建和修改 DashboardState。

**核心设计：State 存储 Code，不存储 Data。** 每个 Panel 的 `code` 字段包含一段 JS 代码字符串，在渲染时由后端沙箱执行以获取最新数据。Agent 写代码，不写数据。

## 完整类型定义

```go
// DashboardState 看板完整状态
type DashboardState struct {
    ID        string            `json:"id"`
    Title     string            `json:"title"`
    CreatedAt time.Time         `json:"createdAt"`
    UpdatedAt time.Time         `json:"updatedAt"`
    Version   int               `json:"version"`       // 每次 mutate +1，用于冲突检测和 undo
    Layout    LayoutConfig      `json:"layout"`
    Panels    []Panel           `json:"panels"`
    GlobalConfig *GlobalConfig  `json:"globalConfig,omitempty"` // 全局配置
    Metadata  map[string]any    `json:"metadata,omitempty"`     // 扩展字段
}

// LayoutConfig 布局配置
type LayoutConfig struct {
    Mode    string      `json:"mode"`              // "grid" | "freeform"
    Columns int         `json:"columns"`           // grid 模式列数
    Gap     int         `json:"gap,omitempty"`     // panel 间距 px
    Rows    []RowConfig `json:"rows,omitempty"`    // 可选，精细行高控制
}

type RowConfig struct {
    Height    string `json:"height"`    // "auto" | "300px" | "1fr"
    MinHeight string `json:"minHeight,omitempty"`
}

// Panel 单个面板
type Panel struct {
    ID         string          `json:"id"`                       // 唯一标识
    Title      string          `json:"title"`
    Type       PanelType       `json:"type"`
    Renderer   RendererType    `json:"renderer"`
    Position   *Position       `json:"position,omitempty"`       // 布局位置
    Code       string          `json:"code"`                     // JS 代码字符串，渲染时执行以获取数据
    Indicators []Indicator     `json:"indicators,omitempty"`     // 技术指标
    Config     map[string]any  `json:"config,omitempty"`         // 渲染器特定配置
    DataSource *DataSourceRef  `json:"dataSource,omitempty"`     // 可选元数据，描述数据来源
    RefreshInterval string     `json:"refreshInterval,omitempty"` // 自动刷新间隔
    LastRefresh *time.Time     `json:"lastRefresh,omitempty"`
}

type PanelType string

const (
    PanelCandlestick PanelType = "candlestick"
    PanelLine        PanelType = "line"
    PanelArea        PanelType = "area"
    PanelBar         PanelType = "bar"
    PanelHeatmap     PanelType = "heatmap"
    PanelPie         PanelType = "pie"
    PanelRadar       PanelType = "radar"
    PanelScatter     PanelType = "scatter"
    PanelTreemap     PanelType = "treemap"
    PanelTable       PanelType = "table"
    PanelFeed        PanelType = "feed"
    PanelMetricCard  PanelType = "metric_card"
    PanelCorrelation PanelType = "correlation"
    PanelSankey      PanelType = "sankey"
)

type RendererType string

const (
    RendererLightweight RendererType = "lightweight-charts"
    RendererECharts     RendererType = "echarts"
    RendererCustom      RendererType = "custom"
)

// Position Grid 布局位置
type Position struct {
    Col     int `json:"col"`               // 起始列 (0-indexed)
    Row     int `json:"row"`               // 起始行
    ColSpan int `json:"colSpan,omitempty"` // 占几列，默认 1
    RowSpan int `json:"rowSpan,omitempty"` // 占几行，默认 1
    Width   int `json:"width,omitempty"`   // freeform 模式: px
    Height  int `json:"height,omitempty"`  // freeform 模式: px
}

// Indicator 技术指标
type Indicator struct {
    Type    string         `json:"type"`              // MA, EMA, SMA, BOLL, RSI, MACD, KDJ, ATR, OBV...
    Period  int            `json:"period,omitempty"`
    Periods []int          `json:"periods,omitempty"` // 多周期（如多条均线）
    Params  map[string]any `json:"params,omitempty"`  // 指标特定参数
    Style   *IndicatorStyle `json:"style,omitempty"`
}

type IndicatorStyle struct {
    Color     string  `json:"color,omitempty"`
    LineWidth float64 `json:"lineWidth,omitempty"`
    Opacity   float64 `json:"opacity,omitempty"`
}

// DataSourceRef 数据来源元数据（可选，用于 UI 展示来源信息）
type DataSourceRef struct {
    Source string         `json:"source"` // SDK 名称
    Method string         `json:"method"` // 方法名
    Params map[string]any `json:"params"` // 调用参数（仅元数据，实际获取逻辑在 panel.code 中）
}

// GlobalConfig 全局配置
type GlobalConfig struct {
    Theme      string `json:"theme,omitempty"`      // "dark" | "light"
    TimeZone   string `json:"timeZone,omitempty"`   // 时区
    Currency   string `json:"currency,omitempty"`   // 货币单位
    DateFormat string `json:"dateFormat,omitempty"` // 日期格式
    Locale     string `json:"locale,omitempty"`     // 语言
}
```

## Panel Code 字段

`code` 是每个 Panel 的核心字段 — 一段 JS 代码字符串，由后端沙箱在渲染时执行。

**代码环境：**
- 可用 `sdk.<source>.<method>(params)` 调用数据源
- 可用 `await` 和 `Promise.all`
- 可用 `utils.*` 工具函数
- 返回值即为该 panel 的渲染数据

**示例 — K线 panel.code：**
```javascript
"const data = await sdk.market.getKline({ symbol: 'XAUUSD', timeframe: '1D', limit: 120 });\nreturn data;"
```

**示例 — 相关性热力图 panel.code：**
```javascript
"const [gold, silver, spx, dxy] = await Promise.all([\n  sdk.market.getDaily({ symbol: 'XAUUSD', limit: 252 }),\n  sdk.market.getDaily({ symbol: 'XAGUSD', limit: 252 }),\n  sdk.market.getDaily({ symbol: 'SPX', limit: 252 }),\n  sdk.market.getDaily({ symbol: 'DXY', limit: 252 })\n]);\nconst closes = [gold, silver, spx, dxy].map(d => d.map(p => p.close));\nreturn utils.computeCorrelationMatrix(closes, ['XAUUSD', 'XAGUSD', 'SPX', 'DXY']);"
```

**示例 — 信息流 panel.code：**
```javascript
"const tweets = await sdk.social.searchTweets({ query: 'gold silver', limit: 50, sentiment: true });\nreturn { items: tweets };"
```

**关键特性：**
- Code 是自包含的 — 包含完整的数据获取逻辑
- 刷新 = 重新执行 code — 无需 Agent 参与
- Agent 编写 code 但不看到执行结果 — 后端通过自动验证反馈成功/失败

## Panel 渲染数据格式规范

panel.code 执行后返回的数据需要符合以下格式（按 PanelType）：

### Candlestick / Line / Area (时序图表)

```json
[
  { "time": "2024-01-15", "open": 2050.3, "high": 2065.1, "low": 2045.2, "close": 2058.7, "volume": 185000 },
  { "time": "2024-01-16", "open": 2058.7, "high": 2072.4, "low": 2055.0, "close": 2068.3, "volume": 203000 }
]
```

Line/Area 可以简化为：
```json
[
  { "time": "2024-01-15", "value": 2058.7 },
  { "time": "2024-01-16", "value": 2068.3 }
]
```

多系列叠加：
```json
{
  "series": [
    { "name": "黄金", "data": [{ "time": "2024-01-15", "value": 2058.7 }] },
    { "name": "白银", "data": [{ "time": "2024-01-15", "value": 23.15 }] }
  ]
}
```

### Bar (柱状图)

```json
{
  "categories": ["Q1", "Q2", "Q3", "Q4"],
  "series": [
    { "name": "营收", "values": [125.3, 132.7, 128.9, 145.2] },
    { "name": "利润", "values": [28.1, 31.5, 27.3, 35.8] }
  ]
}
```

### Heatmap (热力图)

```json
{
  "xLabels": ["XAUUSD", "BTCUSD", "SPX", "DXY"],
  "yLabels": ["XAUUSD", "BTCUSD", "SPX", "DXY"],
  "values": [
    [1.0, 0.35, -0.12, -0.45],
    [0.35, 1.0, 0.28, -0.31],
    [-0.12, 0.28, 1.0, 0.15],
    [-0.45, -0.31, 0.15, 1.0]
  ]
}
```

### Pie (饼图)

```json
[
  { "name": "黄金ETF", "value": 45.2 },
  { "name": "白银ETF", "value": 12.8 },
  { "name": "铜ETF", "value": 8.3 },
  { "name": "其他", "value": 33.7 }
]
```

### Feed (信息流)

```json
{
  "items": [
    {
      "id": "tw_123",
      "type": "tweet",
      "author": { "name": "Peter Schiff", "handle": "@PeterSchiff", "avatar": "..." },
      "text": "Gold breaking through $2100 resistance...",
      "timestamp": "2024-01-16T14:32:00Z",
      "metrics": { "likes": 1523, "retweets": 432 },
      "sentiment": "positive",
      "sentimentScore": 0.82
    }
  ]
}
```

### Table (表格)

```json
{
  "columns": [
    { "key": "symbol", "title": "代码", "type": "string" },
    { "key": "price", "title": "价格", "type": "number", "format": "0,0.00" },
    { "key": "change", "title": "涨跌%", "type": "percent", "colorCoded": true },
    { "key": "volume", "title": "成交量", "type": "number", "format": "0.0a" }
  ],
  "rows": [
    { "symbol": "XAUUSD", "price": 2058.7, "change": 0.0125, "volume": 185000 },
    { "symbol": "XAGUSD", "price": 23.15, "change": -0.0082, "volume": 42000 }
  ]
}
```

### MetricCard (指标卡片)

```json
{
  "value": 2058.7,
  "label": "黄金现货",
  "unit": "USD",
  "change": 25.4,
  "changePercent": 0.0125,
  "direction": "up",
  "sparkline": [2020, 2035, 2028, 2045, 2058],
  "subtitle": "较昨日收盘"
}
```

## State Summary（摘要模式）

`get_state` 的 summary 模式用于编辑时给 Agent 提供上下文，省略 code 完整内容：

```go
func (s *DashboardState) ToSummary() *DashboardSummary {
    panels := make([]PanelSummary, len(s.Panels))
    for i, p := range s.Panels {
        panels[i] = PanelSummary{
            ID:          p.ID,
            Title:       p.Title,
            Type:        p.Type,
            Renderer:    p.Renderer,
            Indicators:  summarizeIndicators(p.Indicators),
            DataSource:  p.DataSource,
            Config:      summarizeConfig(p.Config),
            CodePreview: truncateCode(p.Code, 200), // 截断 code 预览
        }
    }
    return &DashboardSummary{
        ID:     s.ID,
        Title:  s.Title,
        Layout: s.Layout,
        Panels: panels,
    }
}

type PanelSummary struct {
    ID          string          `json:"id"`
    Title       string          `json:"title"`
    Type        PanelType       `json:"type"`
    Renderer    RendererType    `json:"renderer"`
    Indicators  []string        `json:"indicators,omitempty"`  // ["MA(20)", "BOLL(20)"]
    DataSource  *DataSourceRef  `json:"dataSource,omitempty"`
    Config      map[string]any  `json:"config,omitempty"`      // 关键配置
    CodePreview string          `json:"codePreview,omitempty"` // 截断的 code 预览
}
```

Summary 输出示例：
```json
{
  "id": "db_abc123",
  "title": "黄金分析看板",
  "layout": { "mode": "grid", "columns": 2 },
  "panels": [
    {
      "id": "gold-kline",
      "title": "黄金日线",
      "type": "candlestick",
      "renderer": "lightweight-charts",
      "indicators": ["MA(20)", "MA(60)"],
      "dataSource": { "source": "market", "method": "getKline", "params": { "symbol": "XAUUSD", "timeframe": "1D" } },
      "codePreview": "const data = await sdk.market.getKline({ symbol: 'XAUUSD', timeframe: '1D', limit: 120 }); return data;"
    },
    {
      "id": "gold-cot",
      "title": "黄金 COT 持仓",
      "type": "bar",
      "renderer": "echarts",
      "dataSource": { "source": "cot", "method": "getCOTReport", "params": { "symbol": "GOLD" } },
      "codePreview": "const report = await sdk.cot.getCOTReport({ symbol: 'GOLD' }); return { categories: report.map(..."
    },
    {
      "id": "gold-tweets",
      "title": "黄金推特舆情",
      "type": "feed",
      "renderer": "custom",
      "config": { "showSentiment": true, "sortBy": "time" },
      "codePreview": "const tweets = await sdk.social.searchTweets({ query: 'gold', limit: 50 }); return { items: tweets };"
    }
  ]
}
```

## 文件系统状态存储

State 以文件形式持久化：

```
data/users/{user_id}/dashboards/{dashboard_id}/
  state.json          # 当前状态（包含所有 panel 的 code）
  versions/{ts}.json  # 历史版本快照
  meta.json           # 看板元数据（标题、创建时间等）
```

```go
type FileStateStore struct {
    basePath string // "data/users"
}

// Save 保存当前状态并创建版本快照
func (s *FileStateStore) Save(ctx context.Context, userID string, state *DashboardState) error {
    state.Version++
    state.UpdatedAt = time.Now()
    
    dir := filepath.Join(s.basePath, userID, "dashboards", state.ID)
    os.MkdirAll(filepath.Join(dir, "versions"), 0755)
    
    data, _ := json.MarshalIndent(state, "", "  ")
    
    // 写入当前状态
    os.WriteFile(filepath.Join(dir, "state.json"), data, 0644)
    
    // 写入版本快照
    ts := state.UpdatedAt.Format("20060102T150405")
    os.WriteFile(filepath.Join(dir, "versions", ts+".json"), data, 0644)
    
    // 更新 meta
    meta := map[string]any{
        "title":     state.Title,
        "createdAt": state.CreatedAt,
        "updatedAt": state.UpdatedAt,
        "version":   state.Version,
    }
    metaData, _ := json.MarshalIndent(meta, "", "  ")
    os.WriteFile(filepath.Join(dir, "meta.json"), metaData, 0644)
    
    return nil
}

// Get 读取当前状态
func (s *FileStateStore) Get(ctx context.Context, userID, dashboardID string) (*DashboardState, error) {
    path := filepath.Join(s.basePath, userID, "dashboards", dashboardID, "state.json")
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, err
    }
    var state DashboardState
    json.Unmarshal(data, &state)
    return &state, nil
}

// GetVersion 获取特定版本（用于 undo）
func (s *FileStateStore) GetVersion(ctx context.Context, userID, dashboardID, ts string) (*DashboardState, error) {
    path := filepath.Join(s.basePath, userID, "dashboards", dashboardID, "versions", ts+".json")
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, err
    }
    var state DashboardState
    json.Unmarshal(data, &state)
    return &state, nil
}
```

## 版本管理

每次 mutate 操作创建新版本快照：

```go
// ListVersions 列出所有版本
func (s *FileStateStore) ListVersions(ctx context.Context, userID, dashboardID string) ([]string, error) {
    dir := filepath.Join(s.basePath, userID, "dashboards", dashboardID, "versions")
    entries, err := os.ReadDir(dir)
    if err != nil {
        return nil, err
    }
    var versions []string
    for _, e := range entries {
        if strings.HasSuffix(e.Name(), ".json") {
            versions = append(versions, strings.TrimSuffix(e.Name(), ".json"))
        }
    }
    return versions, nil
}
```

## Diff 计算

mutate 后对比新旧 state，生成增量更新：

```go
type StateDiff struct {
    AddedPanels   []Panel       `json:"addedPanels,omitempty"`
    RemovedPanels []string      `json:"removedPanels,omitempty"`   // panel IDs
    UpdatedPanels []PanelUpdate `json:"updatedPanels,omitempty"`
    LayoutChanged bool          `json:"layoutChanged"`
    TitleChanged  bool          `json:"titleChanged"`
}

type PanelUpdate struct {
    PanelID      string         `json:"panelId"`
    ChangedFields []string      `json:"changedFields"` // ["code", "indicators", "config", "title"]
    Panel        Panel          `json:"panel"`          // 完整的新 panel 数据
}

func ComputeDiff(old, new *DashboardState) *StateDiff {
    diff := &StateDiff{}
    
    oldMap := make(map[string]*Panel)
    for i := range old.Panels {
        oldMap[old.Panels[i].ID] = &old.Panels[i]
    }
    
    newMap := make(map[string]*Panel)
    for i := range new.Panels {
        newMap[new.Panels[i].ID] = &new.Panels[i]
    }
    
    // 新增
    for id, p := range newMap {
        if _, exists := oldMap[id]; !exists {
            diff.AddedPanels = append(diff.AddedPanels, *p)
        }
    }
    
    // 删除
    for id := range oldMap {
        if _, exists := newMap[id]; !exists {
            diff.RemovedPanels = append(diff.RemovedPanels, id)
        }
    }
    
    // 修改
    for id, newPanel := range newMap {
        if oldPanel, exists := oldMap[id]; exists {
            changed := detectChangedFields(oldPanel, newPanel)
            if len(changed) > 0 {
                diff.UpdatedPanels = append(diff.UpdatedPanels, PanelUpdate{
                    PanelID:       id,
                    ChangedFields: changed,
                    Panel:         *newPanel,
                })
            }
        }
    }
    
    diff.LayoutChanged = !reflect.DeepEqual(old.Layout, new.Layout)
    diff.TitleChanged = old.Title != new.Title
    
    return diff
}
```
