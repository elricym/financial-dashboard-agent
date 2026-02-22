# 04 - DashboardState 状态模型

## 概述

DashboardState 是整个系统的核心数据结构，是前端渲染的 single source of truth。Agent 生成和编辑看板的本质就是构建和修改 DashboardState。

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
    Data       any             `json:"data"`                     // 渲染数据
    Indicators []Indicator     `json:"indicators,omitempty"`     // 技术指标
    Config     map[string]any  `json:"config,omitempty"`         // 渲染器特定配置
    DataSource *DataSourceRef  `json:"dataSource,omitempty"`     // 数据来源引用
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

// DataSourceRef 数据来源引用（用于刷新）
type DataSourceRef struct {
    Source string         `json:"source"` // SDK 名称
    Method string         `json:"method"` // 方法名
    Params map[string]any `json:"params"` // 调用参数
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

## Panel Data 格式规范

不同 PanelType 对 data 字段的格式要求：

### Candlestick / Line / Area (时序图表)

```json
{
  "data": [
    { "time": "2024-01-15", "open": 2050.3, "high": 2065.1, "low": 2045.2, "close": 2058.7, "volume": 185000 },
    { "time": "2024-01-16", "open": 2058.7, "high": 2072.4, "low": 2055.0, "close": 2068.3, "volume": 203000 }
  ]
}
```

Line/Area 可以简化为：
```json
{
  "data": [
    { "time": "2024-01-15", "value": 2058.7 },
    { "time": "2024-01-16", "value": 2068.3 }
  ]
}
```

多系列叠加：
```json
{
  "data": {
    "series": [
      { "name": "黄金", "data": [{ "time": "2024-01-15", "value": 2058.7 }] },
      { "name": "白银", "data": [{ "time": "2024-01-15", "value": 23.15 }] }
    ]
  }
}
```

### Bar (柱状图)

```json
{
  "data": {
    "categories": ["Q1", "Q2", "Q3", "Q4"],
    "series": [
      { "name": "营收", "values": [125.3, 132.7, 128.9, 145.2] },
      { "name": "利润", "values": [28.1, 31.5, 27.3, 35.8] }
    ]
  }
}
```

### Heatmap (热力图)

```json
{
  "data": {
    "xLabels": ["XAUUSD", "BTCUSD", "SPX", "DXY"],
    "yLabels": ["XAUUSD", "BTCUSD", "SPX", "DXY"],
    "values": [
      [1.0, 0.35, -0.12, -0.45],
      [0.35, 1.0, 0.28, -0.31],
      [-0.12, 0.28, 1.0, 0.15],
      [-0.45, -0.31, 0.15, 1.0]
    ]
  }
}
```

### Pie (饼图)

```json
{
  "data": [
    { "name": "黄金ETF", "value": 45.2 },
    { "name": "白银ETF", "value": 12.8 },
    { "name": "铜ETF", "value": 8.3 },
    { "name": "其他", "value": 33.7 }
  ]
}
```

### Feed (信息流)

```json
{
  "data": {
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
  },
  "config": {
    "sortBy": "time",
    "groupBy": null,
    "showSentiment": true,
    "maxItems": 50
  }
}
```

### Table (表格)

```json
{
  "data": {
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
  },
  "config": {
    "sortable": true,
    "defaultSort": { "key": "change", "order": "desc" },
    "pagination": false
  }
}
```

### MetricCard (指标卡片)

```json
{
  "data": {
    "value": 2058.7,
    "label": "黄金现货",
    "unit": "USD",
    "change": 25.4,
    "changePercent": 0.0125,
    "direction": "up",
    "sparkline": [2020, 2035, 2028, 2045, 2058],
    "subtitle": "较昨日收盘"
  }
}
```

## State Summary（摘要模式）

`get_state` 的 summary 模式用于编辑时给 Agent 提供上下文，省略大块 data：

```go
func (s *DashboardState) ToSummary() *DashboardSummary {
    panels := make([]PanelSummary, len(s.Panels))
    for i, p := range s.Panels {
        panels[i] = PanelSummary{
            ID:         p.ID,
            Title:      p.Title,
            Type:       p.Type,
            Renderer:   p.Renderer,
            Indicators: summarizeIndicators(p.Indicators),
            DataSource: p.DataSource,
            Config:     summarizeConfig(p.Config),
            DataStats:  computeDataStats(p.Data, p.Type), // 数据摘要统计
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
    ID         string          `json:"id"`
    Title      string          `json:"title"`
    Type       PanelType       `json:"type"`
    Renderer   RendererType    `json:"renderer"`
    Indicators []string        `json:"indicators,omitempty"`  // ["MA(20)", "BOLL(20)"]
    DataSource *DataSourceRef  `json:"dataSource,omitempty"`
    Config     map[string]any  `json:"config,omitempty"`      // 关键配置
    DataStats  *DataStats      `json:"dataStats,omitempty"`   // 数据量统计
}

type DataStats struct {
    RowCount  int    `json:"rowCount,omitempty"`  // 数据行数
    TimeRange string `json:"timeRange,omitempty"` // "2024-01-01 ~ 2024-06-30"
    ItemCount int    `json:"itemCount,omitempty"` // feed/table 的条目数
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
      "dataStats": { "rowCount": 120, "timeRange": "2023-09-01 ~ 2024-01-16" }
    },
    {
      "id": "gold-cot",
      "title": "黄金 COT 持仓",
      "type": "bar",
      "renderer": "echarts",
      "dataSource": { "source": "cot", "method": "getCOTReport", "params": { "symbol": "GOLD" } },
      "dataStats": { "rowCount": 52, "timeRange": "2023-01-20 ~ 2024-01-12" }
    },
    {
      "id": "gold-tweets",
      "title": "黄金推特舆情",
      "type": "feed",
      "renderer": "custom",
      "config": { "showSentiment": true, "sortBy": "time" },
      "dataStats": { "itemCount": 50 }
    }
  ]
}
```

## 版本管理

每次 mutate 操作创建新版本：

```go
type StateStore struct {
    redis *redis.Client
    pg    *pgxpool.Pool
}

// Save 保存状态（Redis 热存储 + PG 持久化）
func (s *StateStore) Save(ctx context.Context, state *DashboardState) error {
    state.Version++
    state.UpdatedAt = time.Now()
    
    // Redis: 存最新版本（热数据）
    data, _ := json.Marshal(state)
    s.redis.Set(ctx, fmt.Sprintf("dashboard:%s:current", state.ID), data, 24*time.Hour)
    
    // PG: 存版本历史
    _, err := s.pg.Exec(ctx,
        `INSERT INTO dashboard_versions (dashboard_id, version, state, created_at)
         VALUES ($1, $2, $3, $4)`,
        state.ID, state.Version, data, state.UpdatedAt,
    )
    return err
}

// GetVersion 获取特定版本（用于 undo）
func (s *StateStore) GetVersion(ctx context.Context, dashboardID string, version int) (*DashboardState, error) {
    var data []byte
    err := s.pg.QueryRow(ctx,
        `SELECT state FROM dashboard_versions WHERE dashboard_id = $1 AND version = $2`,
        dashboardID, version,
    ).Scan(&data)
    if err != nil {
        return nil, err
    }
    var state DashboardState
    json.Unmarshal(data, &state)
    return &state, nil
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
    ChangedFields []string      `json:"changedFields"` // ["data", "indicators", "config", "title"]
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
