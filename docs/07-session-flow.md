# 07 - 会话与编辑流程

## 会话生命周期

```
用户连接 → 创建/恢复 Session → 对话循环 → 断开(保留 Session)
```

### Session 结构

```go
type Session struct {
    ID                string          `json:"id"`
    UserID            string          `json:"userId"`
    ActiveDashboardID string          `json:"activeDashboardId,omitempty"`
    DashboardIDs      []string        `json:"dashboardIds"`      // 所有看板
    Messages          []eino.Message  `json:"messages"`          // 对话历史
    CreatedAt         time.Time       `json:"createdAt"`
    LastActiveAt      time.Time       `json:"lastActiveAt"`
}
```

## 工作流程总览

系统有两条独立路径：**编辑路径**（需要 Agent）和**刷新路径**（不需要 Agent）。

```
┌─────────────────────────────────────────────────────────────────┐
│                     系统工作流程                                   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐        │
│  │              编辑路径 (Edit Path)                     │        │
│  │                                                     │        │
│  │  用户自然语言 → Agent 理解意图                         │        │
│  │       │                                             │        │
│  │       ▼                                             │        │
│  │  Agent 调用 execute/mutate                           │        │
│  │  → 生成/修改 state.json (含 panel.code)               │        │
│  │       │                                             │        │
│  │       ▼                                             │        │
│  │  ┌───────────────────────────┐                      │        │
│  │  │  自动验证 (Auto-Validate) │                      │        │
│  │  │  试运行所有 panel.code     │                      │        │
│  │  │  返回执行摘要给 Agent      │                      │        │
│  │  └─────────┬─────────────────┘                      │        │
│  │            │                                        │        │
│  │     成功？ ├── 是 → 持久化 + 推送前端                  │        │
│  │            └── 否 → Agent 用 mutate 修复 code         │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐        │
│  │              刷新路径 (Refresh Path)                  │        │
│  │                                                     │        │
│  │  用户点击刷新 / 自动刷新定时器                          │        │
│  │       │                                             │        │
│  │       ▼                                             │        │
│  │  后端重新执行 panel.code（无需 Agent）                  │        │
│  │       │                                             │        │
│  │       ▼                                             │        │
│  │  新 data → WebSocket 推送 → 前端更新                   │        │
│  └─────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

**关键洞察：** Agent 写代码字符串，不看到数据结果。后端验证机制提供 success/fail/row count 反馈，Agent 据此自修复。

## 完整对话流程

### 场景 1：首次生成看板

```
用户: "帮我做一个黄金和白银的对比看板，要K线对比、相关性热力图和最新舆情"

→ Session Manager 构建 messages:
  [system] System Prompt + "当前无活跃看板"
  [user] "帮我做一个黄金和白银的对比看板..."

→ Eino Agent 处理:
  
  Step 1 - Agent 思考:
    "用户需要三个面板：黄金白银K线对比、相关性热力图、舆情。
     需要搜索行情数据源和舆情数据源。"
  
  → WS 推送: { type: 'agent:thinking', payload: { message: '分析需求中...' } }
  
  Step 2 - Agent 调用 search:
    search({
      code: `
        var results = [];
        spec.sources.forEach(function(src) {
          src.methods.forEach(function(m) {
            if (m.name === 'getKline' || m.name === 'getDaily' ||
                m.name === 'searchTweets' || m.name === 'getSentimentTimeline') {
              results.push({ source: src.name, method: m.name, description: m.description, params: m.params });
            }
          });
        });
        results;
      `
    })
    
  → 返回: [
      { source: "market", method: "getKline", ... },
      { source: "market", method: "getDaily", ... },
      { source: "social", method: "searchTweets", ... },
    ]
  
  Step 3 - Agent 调用 execute:
    execute({
      code: `
        (function() {
          return {
            title: '黄金 vs 白银 对比看板',
            layout: { mode: 'grid', columns: 2, gap: 8 },
            panels: [
              {
                id: 'gold-kline',
                title: '黄金日线 XAUUSD',
                type: 'candlestick',
                renderer: 'lightweight-charts',
                position: { col: 0, row: 0 },
                code: "const data = await sdk.market.getKline({ symbol: 'XAUUSD', timeframe: '1D', limit: 120 }); return data;",
                indicators: [{ type: 'MA', periods: [20, 60] }],
                dataSource: { source: 'market', method: 'getKline', params: { symbol: 'XAUUSD', timeframe: '1D' } }
              },
              {
                id: 'silver-kline',
                title: '白银日线 XAGUSD',
                type: 'candlestick',
                renderer: 'lightweight-charts',
                position: { col: 1, row: 0 },
                code: "const data = await sdk.market.getKline({ symbol: 'XAGUSD', timeframe: '1D', limit: 120 }); return data;",
                indicators: [{ type: 'MA', periods: [20, 60] }],
                dataSource: { source: 'market', method: 'getKline', params: { symbol: 'XAGUSD', timeframe: '1D' } }
              },
              {
                id: 'correlation',
                title: '黄金白银相关性',
                type: 'heatmap',
                renderer: 'echarts',
                position: { col: 0, row: 1 },
                code: "const [gold, silver] = await Promise.all([sdk.market.getDaily({ symbol: 'XAUUSD', limit: 252 }), sdk.market.getDaily({ symbol: 'XAGUSD', limit: 252 })]); const closes = [gold, silver].map(d => d.map(p => p.close)); return utils.computeCorrelationMatrix(closes, ['XAUUSD', 'XAGUSD']);",
                config: { colorScale: 'RdYlGn' }
              },
              {
                id: 'precious-metals-feed',
                title: '贵金属舆情',
                type: 'feed',
                renderer: 'custom',
                position: { col: 1, row: 1 },
                code: "const tweets = await sdk.social.searchTweets({ query: 'gold silver', limit: 50, sentiment: true }); return { items: tweets };",
                config: { showSentiment: true, sortBy: 'time' },
                dataSource: { source: 'social', method: 'searchTweets', params: { query: 'gold silver', limit: 50 } }
              }
            ]
          };
        })()
      `
    })

  → 后端执行 Agent 代码 → 得到 DashboardState（含 panel.code）
  → 自动验证：试运行每个 panel.code
  → 返回验证结果:
    { allOk: true, panels: [
      { panelId: "gold-kline", status: "success", rowCount: 120 },
      { panelId: "silver-kline", status: "success", rowCount: 120 },
      { panelId: "correlation", status: "success", rowCount: 4 },
      { panelId: "precious-metals-feed", status: "success", rowCount: 50 }
    ]}
  → State Store 持久化 state.json
  → 后端执行所有 panel.code → 获取 data → WebSocket 推送前端渲染
  
  Step 4 - Agent 回复:
    "已为您生成「黄金 vs 白银 对比看板」，包含：
     - 黄金日线K线（含MA20/60）— 120条数据
     - 白银日线K线（含MA20/60）— 120条数据
     - 黄金白银相关性热力图
     - 贵金属推特舆情（含情绪分析）— 50条
     
     需要调整可以随时说。"
```

### 场景 2：编辑看板

```
用户: "把K线都换成周线，相关性那个加上标普和美元指数"

→ Session Manager 构建 messages:
  [system] System Prompt + "当前看板: 黄金 vs 白银 对比看板 (db_xxx), panels: gold-kline, silver-kline, correlation, precious-metals-feed"
  [history] ... 之前的对话 ...
  [user] "把K线都换成周线，相关性那个加上标普和美元指数"

→ Eino Agent 处理:

  Step 1 - Agent 调用 get_state (summary):
    get_state({ detail_level: "summary" })
    
  → 返回当前看板摘要（含 codePreview）

  Step 2 - Agent 调用 mutate:
    mutate({
      code: `
        (function(state) {
          // 1. K线换周线 — 修改 code 字符串
          var goldPanel = state.panels.find(function(p) { return p.id === 'gold-kline'; });
          goldPanel.title = '黄金周线 XAUUSD';
          goldPanel.code = "const data = await sdk.market.getKline({ symbol: 'XAUUSD', timeframe: '1W', limit: 120 }); return data;";
          goldPanel.dataSource.params.timeframe = '1W';
          
          var silverPanel = state.panels.find(function(p) { return p.id === 'silver-kline'; });
          silverPanel.title = '白银周线 XAGUSD';
          silverPanel.code = "const data = await sdk.market.getKline({ symbol: 'XAGUSD', timeframe: '1W', limit: 120 }); return data;";
          silverPanel.dataSource.params.timeframe = '1W';
          
          // 2. 更新相关性矩阵 code：加入 SPX 和 DXY
          var corrPanel = state.panels.find(function(p) { return p.id === 'correlation'; });
          corrPanel.title = '多资产相关性矩阵';
          corrPanel.code = "const [gold, silver, spx, dxy] = await Promise.all([" +
            "sdk.market.getDaily({ symbol: 'XAUUSD', limit: 252 })," +
            "sdk.market.getDaily({ symbol: 'XAGUSD', limit: 252 })," +
            "sdk.market.getDaily({ symbol: 'SPX', limit: 252 })," +
            "sdk.market.getDaily({ symbol: 'DXY', limit: 252 })" +
            "]); const closes = [gold, silver, spx, dxy].map(d => d.map(p => p.close));" +
            "return utils.computeCorrelationMatrix(closes, ['XAUUSD', 'XAGUSD', 'SPX', 'DXY']);";
          
          // 调整布局
          corrPanel.position = { col: 0, row: 1, colSpan: 2 };
          state.panels.find(function(p) { return p.id === 'precious-metals-feed'; }).position = { col: 0, row: 2, colSpan: 2 };
          
          return state;
        })
      `
    })

  → 后端执行 Agent 的 mutate 代码 → 得到新 state
  → Diff 计算: updatedPanels: [gold-kline, silver-kline, correlation, precious-metals-feed(position)]
  → 自动验证变更的 panel code → 返回验证结果
  → State Store 更新 state.json + 版本快照
  → 后端重新执行变更 panel 的 code → 新 data → WebSocket 推送
  → 前端局部刷新
  
  Step 3 - Agent 回复:
    "已更新：
     - K线已切换为周线
     - 相关性矩阵加入了标普500和美元指数
     - 布局已调整，相关性面板扩展为全宽"
```

### 场景 3：自由发挥的需求

```
用户: "帮我加一个面板，显示最近一年黄金和比特币的走势叠加图，都标准化到0-100"

→ Agent 调用 mutate:
  mutate({
    code: `
      (function(state) {
        state.panels.push({
          id: 'gold-btc-overlay',
          title: '黄金 vs 比特币 标准化走势 (0-100)',
          type: 'line',
          renderer: 'lightweight-charts',
          code: "const [gold, btc] = await Promise.all([" +
            "sdk.market.getDaily({ symbol: 'XAUUSD', limit: 252 })," +
            "sdk.market.getDaily({ symbol: 'BTCUSD', limit: 252 })" +
            "]);" +
            "const goldNorm = utils.normalize(gold.map(d => d.close)).map(v => v * 100);" +
            "const btcNorm = utils.normalize(btc.map(d => d.close)).map(v => v * 100);" +
            "const times = gold.map(d => d.date);" +
            "return { series: [" +
            "  { name: '黄金', data: times.map((t, i) => ({ time: t, value: goldNorm[i] })) }," +
            "  { name: '比特币', data: times.map((t, i) => ({ time: t, value: btcNorm[i] })) }" +
            "]};",
          config: { multiSeries: true }
        });
        
        return state;
      })
    `
  })
  
  → 自动验证: { panelId: "gold-btc-overlay", status: "success", rowCount: 252 }
```

### 场景 4：复杂分析需求

```
用户: "帮我做一个宏观仪表盘，要看美国CPI趋势、10Y-2Y利差、非农就业，加上下周的经济日历"

→ Agent 调用 search (找到 macro 数据源)
→ Agent 调用 execute:
  execute({
    code: `
      (function() {
        return {
          title: '美国宏观经济仪表盘',
          layout: { mode: 'grid', columns: 2, gap: 8 },
          panels: [
            {
              id: 'us-cpi',
              title: '美国 CPI 同比',
              type: 'bar',
              renderer: 'echarts',
              position: { col: 0, row: 0 },
              code: "const data = await sdk.macro.getIndicator({ indicator: 'CPI', country: 'US', limit: 24 }); return { categories: data.map(d => d.date), series: [{ name: 'CPI YoY%', values: data.map(d => d.value) }] };",
              config: { colorByValue: true, threshold: 2.0 },
              dataSource: { source: 'macro', method: 'getIndicator', params: { indicator: 'CPI', country: 'US' } }
            },
            {
              id: 'yield-spread',
              title: '10Y-2Y 利差',
              type: 'area',
              renderer: 'lightweight-charts',
              position: { col: 1, row: 0 },
              code: "const rates = await sdk.macro.getInterestRates({ country: 'US', types: ['10Y', '2Y', 'spread_10Y_2Y'] }); return rates.spread_10Y_2Y.map(d => ({ time: d.date, value: d.value }));",
              config: { zeroLine: true, positiveColor: 'rgba(38,166,154,0.4)', negativeColor: 'rgba(239,83,80,0.4)' },
              dataSource: { source: 'macro', method: 'getInterestRates', params: { country: 'US' } }
            },
            {
              id: 'unemployment',
              title: '失业率',
              type: 'line',
              renderer: 'lightweight-charts',
              position: { col: 0, row: 1 },
              code: "const data = await sdk.macro.getIndicator({ indicator: 'UNEMPLOYMENT', country: 'US', limit: 24 }); return data.map(d => ({ time: d.date, value: d.value }));",
              dataSource: { source: 'macro', method: 'getIndicator', params: { indicator: 'UNEMPLOYMENT', country: 'US' } }
            },
            {
              id: 'econ-calendar',
              title: '本周经济日历',
              type: 'table',
              renderer: 'custom',
              position: { col: 1, row: 1 },
              code: "const events = await sdk.macro.getEconomicCalendar({ days: 7, importance: 'high', country: 'US' }); return { columns: [{ key: 'datetime', title: '时间', type: 'datetime' }, { key: 'event', title: '事件', type: 'string' }, { key: 'importance', title: '重要性', type: 'badge' }, { key: 'forecast', title: '预期', type: 'number' }, { key: 'previous', title: '前值', type: 'number' }], rows: events };",
              dataSource: { source: 'macro', method: 'getEconomicCalendar', params: { days: 7, importance: 'high' } }
            }
          ]
        };
      })()
    `
  })
  
  → 自动验证所有 panel.code → 全部成功
  → 持久化 + 推送前端
```

### 场景 5：自动验证失败和自修复

```
→ Agent execute 返回 state，自动验证结果:
  {
    allOk: false,
    panels: [
      { panelId: "us-cpi", status: "success", rowCount: 24 },
      { panelId: "yield-spread", status: "error", error: "sdk.macro.getInterestRates: unknown type 'spread_10Y_2Y', use 'SPREAD_10Y_2Y'" },
      { panelId: "unemployment", status: "success", rowCount: 24 },
      { panelId: "econ-calendar", status: "success", rowCount: 8 }
    ]
  }

→ Agent 看到验证失败，自动调用 mutate 修复:
  mutate({
    code: `
      (function(state) {
        var panel = state.panels.find(p => p.id === 'yield-spread');
        panel.code = "const rates = await sdk.macro.getInterestRates({ country: 'US', types: ['10Y', '2Y', 'SPREAD_10Y_2Y'] }); return rates.SPREAD_10Y_2Y.map(d => ({ time: d.date, value: d.value }));";
        return state;
      })
    `
  })
  
→ 重新验证 yield-spread → 成功 → 推送前端
```

## 错误恢复

### Agent 代码执行失败

```
用户: "帮我加一个比特币链上MVRV指标"

→ Agent 调用 mutate，但 panel.code 验证时 onchain SDK 返回错误

→ 验证结果返回给 Agent:
  { panelId: "btc-mvrv", status: "error", error: "sdk.onchain.getMetrics: API rate limit exceeded, retry after 60s" }

→ Agent 回复用户:
  "链上数据接口暂时限流了，大约1分钟后可以重试。面板已添加，刷新即可获取数据。"
```

### Agent 生成了无效 State

```
→ Agent execute 代码执行成功，但返回的 state 缺少必需字段

→ Validator 捕获:
  { error: "invalid DashboardState: panel[0] missing 'id' field" }

→ 返回给 Agent，Agent 修正代码重试（最多 2 次）
```

## 上下文注入策略

每轮对话自动在 system prompt 末尾附加当前看板上下文：

```go
func (s *Session) buildDashboardContext() string {
    if s.ActiveDashboardID == "" {
        return "\n\n[当前无活跃看板]"
    }
    
    state, _ := s.stateStore.Get(s.UserID, s.ActiveDashboardID)
    summary := state.ToSummary()
    
    var buf strings.Builder
    buf.WriteString(fmt.Sprintf("\n\n[当前看板: \"%s\" (%s)]\n", state.Title, state.ID))
    buf.WriteString(fmt.Sprintf("布局: %dx grid\n", state.Layout.Columns))
    buf.WriteString("面板:\n")
    for _, p := range summary.Panels {
        buf.WriteString(fmt.Sprintf("  - %s: %s (%s, %s)", p.ID, p.Title, p.Type, p.Renderer))
        if len(p.Indicators) > 0 {
            buf.WriteString(fmt.Sprintf(" [%s]", strings.Join(p.Indicators, ", ")))
        }
        buf.WriteString("\n")
    }
    
    if len(s.DashboardIDs) > 1 {
        buf.WriteString(fmt.Sprintf("\n其他看板: %d 个\n", len(s.DashboardIDs)-1))
    }
    
    return buf.String()
}
```

输出示例：
```
[当前看板: "黄金 vs 白银 对比看板" (db_abc123)]
布局: 2x grid
面板:
  - gold-kline: 黄金周线 XAUUSD (candlestick, lightweight-charts) [MA(20), MA(60)]
  - silver-kline: 白银周线 XAGUSD (candlestick, lightweight-charts) [MA(20), MA(60)]
  - correlation: 多资产相关性矩阵 (heatmap, echarts)
  - precious-metals-feed: 贵金属舆情 (feed, custom)
```
