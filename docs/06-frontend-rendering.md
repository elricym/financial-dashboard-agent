# 06 - 前端渲染层

## 架构概述

```
┌──────────────────────────────────────────────────────────────┐
│                      App Shell                                │
│                                                              │
│  ┌────────────────┐  ┌────────────────────────────────────┐  │
│  │   Chat Panel    │  │        Dashboard Canvas            │  │
│  │                │  │                                    │  │
│  │  ┌────────────┐│  │  ┌──────────────────────────────┐  │  │
│  │  │ Message    ││  │  │      GridStack Layout        │  │  │
│  │  │ List       ││  │  │                              │  │  │
│  │  │            ││  │  │  ┌──────────┐ ┌──────────┐  │  │  │
│  │  │            ││  │  │  │PanelWrap │ │PanelWrap │  │  │  │
│  │  │            ││  │  │  │┌────────┐│ │┌────────┐│  │  │  │
│  │  │            ││  │  │  ││LWChart ││ ││ECharts ││  │  │  │
│  │  │            ││  │  │  │└────────┘│ │└────────┘│  │  │  │
│  │  │            ││  │  │  └──────────┘ └──────────┘  │  │  │
│  │  │            ││  │  │                              │  │  │
│  │  │            ││  │  │  ┌──────────┐ ┌──────────┐  │  │  │
│  │  │            ││  │  │  │PanelWrap │ │PanelWrap │  │  │  │
│  │  │            ││  │  │  │┌────────┐│ │┌────────┐│  │  │  │
│  │  │            ││  │  │  ││Custom  ││ ││LWChart ││  │  │  │
│  │  │            ││  │  │  ││Feed    ││ ││        ││  │  │  │
│  │  └────────────┘│  │  │  │└────────┘│ │└────────┘│  │  │  │
│  │  ┌────────────┐│  │  │  └──────────┘ └──────────┘  │  │  │
│  │  │ Input Box  ││  │  │                              │  │  │
│  │  └────────────┘│  │  └──────────────────────────────┘  │  │
│  └────────────────┘  └────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ Dashboard Tabs: [黄金分析] [加密货币] [+新建]             ││
│  └──────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
```

## 渲染流程：Code → Data → UI

后端负责执行 panel.code 并将数据推送给前端。前端不直接执行 code。

```
┌─────────────────────────────────────────────────────────────────┐
│                        渲染流程                                   │
│                                                                 │
│  state.json (含 panel.code)                                      │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────────────────┐                                │
│  │  后端 Code Executor          │                                │
│  │  for each panel:            │                                │
│  │    sandbox.execute(panel.code)                               │
│  │    → data                   │                                │
│  └──────────┬──────────────────┘                                │
│             │                                                   │
│             ▼                                                   │
│  ┌─────────────────────────────┐                                │
│  │  WebSocket 推送              │                                │
│  │  { panels: [                │                                │
│  │    { id, title, type,       │                                │
│  │      renderer, data, ... }  │   ← data 由后端执行 code 注入    │
│  │  ]}                         │                                │
│  └──────────┬──────────────────┘                                │
│             │                                                   │
│             ▼                                                   │
│  ┌─────────────────────────────┐                                │
│  │  前端渲染                    │                                │
│  │  PanelWrapper → Renderer    │                                │
│  └─────────────────────────────┘                                │
└─────────────────────────────────────────────────────────────────┘
```

**刷新 = 重新执行 code，无需 Agent：**
```
用户点击刷新 → 后端重新执行 panel.code → 新 data → WebSocket 推送 → 前端更新
```

### 后端 Code Executor

```go
type CodeExecutor struct {
    sandboxPool *SandboxPool
    adapters    *CachedAdapterRegistry
}

// ExecuteAllPanels 执行所有 panel 的 code，返回带 data 的渲染数据
func (e *CodeExecutor) ExecuteAllPanels(ctx context.Context, state *DashboardState) ([]PanelRenderData, error) {
    results := make([]PanelRenderData, len(state.Panels))
    
    var wg sync.WaitGroup
    for i, panel := range state.Panels {
        wg.Add(1)
        go func(idx int, p Panel) {
            defer wg.Done()
            
            sandbox := e.sandboxPool.Get()
            defer e.sandboxPool.Put(sandbox)
            
            sandbox.Inject("sdk", e.adapters)
            sandbox.Inject("utils", standardUtils)
            
            data, err := sandbox.Execute(ctx, p.Code)
            
            results[idx] = PanelRenderData{
                ID:         p.ID,
                Title:      p.Title,
                Type:       p.Type,
                Renderer:   p.Renderer,
                Position:   p.Position,
                Data:       data,
                Indicators: p.Indicators,
                Config:     p.Config,
                Error:      errStr(err),
            }
        }(i, panel)
    }
    wg.Wait()
    
    return results, nil
}

type PanelRenderData struct {
    ID         string         `json:"id"`
    Title      string         `json:"title"`
    Type       PanelType      `json:"type"`
    Renderer   RendererType   `json:"renderer"`
    Position   *Position      `json:"position,omitempty"`
    Data       any            `json:"data"`
    Indicators []Indicator    `json:"indicators,omitempty"`
    Config     map[string]any `json:"config,omitempty"`
    Error      string         `json:"error,omitempty"`
}
```

## 组件树

```
App
├── DashboardTabs              // 多看板切换
├── ChatPanel                  // 左侧对话面板
│   ├── MessageList
│   │   ├── UserMessage
│   │   ├── AgentMessage
│   │   └── StatusMessage      // "正在生成看板..." 等状态
│   └── ChatInput
└── DashboardCanvas            // 右侧看板画布
    └── GridLayout             // gridstack.js 包装
        └── PanelWrapper[]     // 每个 panel 的容器
            ├── PanelHeader    // 标题、操作按钮（含刷新按钮）
            ├── PanelContent   // 根据 renderer 分发
            │   ├── LWChartPanel
            │   ├── EChartsPanel
            │   └── CustomPanel
            │       ├── FeedPanel
            │       ├── TablePanel
            │       └── MetricCardPanel
            └── PanelFooter    // 数据来源、刷新时间
```

## WebSocket 通信

### 连接管理

```typescript
class DashboardSocket {
  private ws: WebSocket;
  private handlers: Map<string, Function[]> = new Map();
  
  connect(sessionId: string) {
    this.ws = new WebSocket(`/ws?session=${sessionId}`);
    
    this.ws.onmessage = (event) => {
      const msg: WSMessage = JSON.parse(event.data);
      this.emit(msg.type, msg.payload);
    };
  }
  
  on(type: string, handler: Function) {
    if (!this.handlers.has(type)) this.handlers.set(type, []);
    this.handlers.get(type)!.push(handler);
  }
}
```

### 消息类型

```typescript
type WSMessage =
  | { type: 'dashboard:created'; payload: { dashboardId: string; panels: PanelRenderData[] } }
  | { type: 'dashboard:updated'; payload: { dashboardId: string; diff: StateDiff; panels: PanelRenderData[] } }
  | { type: 'dashboard:deleted'; payload: { dashboardId: string } }
  | { type: 'agent:thinking';    payload: { message: string } }
  | { type: 'agent:executing';   payload: { tool: string; code?: string } }
  | { type: 'agent:message';     payload: { text: string } }
  | { type: 'panel:refreshed';   payload: { dashboardId: string; panelId: string; data: any } }
  | { type: 'error';             payload: { message: string; code?: string } }
```

### 前端状态更新

```typescript
// 前端 store 存储的是带 data 的渲染数据，不是 code
interface DashboardStore {
  dashboards: Map<string, DashboardRenderState>;
  activeDashboardId: string | null;
  
  applyUpdate(dashboardId: string, panels: PanelRenderData[]): void;
  refreshPanel(dashboardId: string, panelId: string): void;
  setActive(dashboardId: string): void;
}

// 刷新单个 panel — 调用后端 API，后端重新执行 panel.code
async function refreshPanel(dashboardId: string, panelId: string) {
  const res = await fetch(`/api/dashboard/${dashboardId}/panel/${panelId}/refresh`, {
    method: 'POST',
  });
  // 新数据通过 WebSocket panel:refreshed 推送
}

// 刷新整个看板
async function refreshDashboard(dashboardId: string) {
  const res = await fetch(`/api/dashboard/${dashboardId}/refresh`, {
    method: 'POST',
  });
  // 所有 panel 的新数据通过 WebSocket 推送
}
```

## 渲染器实现

### PanelWrapper 分发

```tsx
function PanelContent({ panel }: { panel: PanelRenderData }) {
  if (panel.error) {
    return <PanelError error={panel.error} panelId={panel.id} />;
  }
  
  switch (panel.renderer) {
    case 'lightweight-charts':
      return <LWChartPanel panel={panel} />;
    case 'echarts':
      return <EChartsPanel panel={panel} />;
    case 'custom':
      return <CustomPanelRouter panel={panel} />;
    default:
      return <div>Unsupported renderer: {panel.renderer}</div>;
  }
}

function CustomPanelRouter({ panel }: { panel: PanelRenderData }) {
  switch (panel.type) {
    case 'feed':        return <FeedPanel panel={panel} />;
    case 'table':       return <TablePanel panel={panel} />;
    case 'metric_card': return <MetricCardPanel panel={panel} />;
    default:            return <GenericCustomPanel panel={panel} />;
  }
}
```

### Lightweight Charts Panel

```tsx
import { createChart, IChartApi, ISeriesApi } from 'lightweight-charts';

function LWChartPanel({ panel }: { panel: PanelRenderData }) {
  const containerRef = useRef<HTMLDivElement>(null);
  const chartRef = useRef<IChartApi | null>(null);
  const seriesRef = useRef<ISeriesApi<any> | null>(null);
  
  useEffect(() => {
    if (!containerRef.current) return;
    
    const chart = createChart(containerRef.current, {
      layout: {
        background: { color: '#1a1a2e' },
        textColor: '#e0e0e0',
      },
      grid: {
        vertLines: { color: '#2a2a3e' },
        horzLines: { color: '#2a2a3e' },
      },
      timeScale: { timeVisible: true },
      crosshair: { mode: 0 },
    });
    
    let series: ISeriesApi<any>;
    switch (panel.type) {
      case 'candlestick':
        series = chart.addCandlestickSeries({
          upColor: '#26a69a',
          downColor: '#ef5350',
          borderVisible: false,
          wickUpColor: '#26a69a',
          wickDownColor: '#ef5350',
        });
        break;
      case 'line':
        series = chart.addLineSeries({ color: '#2196F3', lineWidth: 2 });
        break;
      case 'area':
        series = chart.addAreaSeries({
          topColor: 'rgba(33, 150, 243, 0.4)',
          bottomColor: 'rgba(33, 150, 243, 0.0)',
          lineColor: '#2196F3',
        });
        break;
    }
    
    series.setData(panel.data);
    
    if (panel.indicators) {
      for (const ind of panel.indicators) {
        addIndicatorToChart(chart, series, panel.data, ind);
      }
    }
    
    chart.timeScale().fitContent();
    chartRef.current = chart;
    seriesRef.current = series;
    
    const ro = new ResizeObserver(() => {
      chart.applyOptions({
        width: containerRef.current!.clientWidth,
        height: containerRef.current!.clientHeight,
      });
    });
    ro.observe(containerRef.current);
    
    return () => { ro.disconnect(); chart.remove(); };
  }, []);
  
  useEffect(() => {
    if (seriesRef.current && panel.data) {
      seriesRef.current.setData(panel.data);
    }
  }, [panel.data]);
  
  return <div ref={containerRef} style={{ width: '100%', height: '100%' }} />;
}
```

### 技术指标叠加

```typescript
function addIndicatorToChart(
  chart: IChartApi,
  mainSeries: ISeriesApi<any>,
  ohlcvData: any[],
  indicator: Indicator
) {
  const closes = ohlcvData.map(d => d.close || d.value);
  const times = ohlcvData.map(d => d.time);
  
  switch (indicator.type) {
    case 'MA':
    case 'SMA': {
      const periods = indicator.periods || [indicator.period!];
      const colors = ['#FF9800', '#9C27B0', '#00BCD4', '#E91E63'];
      periods.forEach((period, i) => {
        const ma = computeMA(closes, period);
        const lineSeries = chart.addLineSeries({
          color: colors[i % colors.length],
          lineWidth: 1,
          priceLineVisible: false,
        });
        lineSeries.setData(
          ma.map((v, idx) => ({ time: times[idx], value: v })).filter(d => d.value !== null)
        );
      });
      break;
    }
    
    case 'BOLL': {
      const period = indicator.period || 20;
      const stdDev = indicator.params?.stdDev || 2;
      const { upper, middle, lower } = computeBOLL(closes, period, stdDev);
      
      chart.addLineSeries({ color: 'rgba(255,152,0,0.5)', lineWidth: 1, priceLineVisible: false })
        .setData(upper.map((v, i) => ({ time: times[i], value: v })).filter(d => d.value !== null));
      chart.addLineSeries({ color: 'rgba(255,152,0,0.8)', lineWidth: 1, priceLineVisible: false })
        .setData(middle.map((v, i) => ({ time: times[i], value: v })).filter(d => d.value !== null));
      chart.addLineSeries({ color: 'rgba(255,152,0,0.5)', lineWidth: 1, priceLineVisible: false })
        .setData(lower.map((v, i) => ({ time: times[i], value: v })).filter(d => d.value !== null));
      break;
    }
    
    case 'RSI': {
      const period = indicator.period || 14;
      const rsi = computeRSI(closes, period);
      const rsiSeries = chart.addLineSeries({
        color: '#E040FB',
        lineWidth: 1,
        priceLineVisible: false,
        priceScaleId: 'rsi',
      });
      rsiSeries.setData(
        rsi.map((v, i) => ({ time: times[i], value: v })).filter(d => d.value !== null)
      );
      chart.priceScale('rsi').applyOptions({
        scaleMargins: { top: 0.8, bottom: 0 },
      });
      break;
    }
  }
}
```

### ECharts Panel

```tsx
import * as echarts from 'echarts';

function EChartsPanel({ panel }: { panel: PanelRenderData }) {
  const containerRef = useRef<HTMLDivElement>(null);
  const chartRef = useRef<echarts.ECharts | null>(null);
  
  useEffect(() => {
    if (!containerRef.current) return;
    
    const chart = echarts.init(containerRef.current, 'dark');
    const option = buildEChartsOption(panel);
    chart.setOption(option);
    chartRef.current = chart;
    
    const ro = new ResizeObserver(() => chart.resize());
    ro.observe(containerRef.current);
    
    return () => { ro.disconnect(); chart.dispose(); };
  }, []);
  
  useEffect(() => {
    if (chartRef.current) {
      chartRef.current.setOption(buildEChartsOption(panel), true);
    }
  }, [panel.data, panel.config]);
  
  return <div ref={containerRef} style={{ width: '100%', height: '100%' }} />;
}

function buildEChartsOption(panel: PanelRenderData): echarts.EChartsOption {
  switch (panel.type) {
    case 'heatmap':  return buildHeatmapOption(panel);
    case 'pie':      return buildPieOption(panel);
    case 'radar':    return buildRadarOption(panel);
    case 'bar':      return buildBarOption(panel);
    case 'scatter':  return buildScatterOption(panel);
    case 'treemap':  return buildTreemapOption(panel);
    case 'sankey':   return buildSankeyOption(panel);
    default:         return {};
  }
}

function buildHeatmapOption(panel: PanelRenderData): echarts.EChartsOption {
  const { xLabels, yLabels, values } = panel.data;
  const data: number[][] = [];
  values.forEach((row: number[], i: number) => {
    row.forEach((val: number, j: number) => {
      data.push([j, i, val]);
    });
  });
  
  return {
    tooltip: { position: 'top' },
    grid: { top: 40, bottom: 40, left: 80, right: 40 },
    xAxis: { type: 'category', data: xLabels, splitArea: { show: true } },
    yAxis: { type: 'category', data: yLabels, splitArea: { show: true } },
    visualMap: {
      min: -1, max: 1,
      calculable: true,
      orient: 'horizontal',
      left: 'center',
      bottom: 0,
      inRange: {
        color: panel.config?.colorScale === 'RdYlGn'
          ? ['#d73027', '#fee08b', '#1a9850']
          : ['#313695', '#ffffbf', '#a50026']
      }
    },
    series: [{
      type: 'heatmap',
      data,
      label: { show: true, formatter: (p: any) => p.data[2].toFixed(2) },
    }]
  };
}
```

### Feed Panel

```tsx
function FeedPanel({ panel }: { panel: PanelRenderData }) {
  const { items } = panel.data;
  const config = panel.config || {};
  
  const sortedItems = useMemo(() => {
    let sorted = [...items];
    switch (config.sortBy) {
      case 'engagement':
        sorted.sort((a, b) => (b.metrics?.likes || 0) - (a.metrics?.likes || 0));
        break;
      case 'sentiment':
        sorted.sort((a, b) => (b.sentimentScore || 0) - (a.sentimentScore || 0));
        break;
      case 'time':
      default:
        sorted.sort((a, b) => new Date(b.timestamp).getTime() - new Date(a.timestamp).getTime());
    }
    return sorted;
  }, [items, config.sortBy]);
  
  return (
    <div className="feed-panel">
      {sortedItems.map(item => (
        <FeedItem key={item.id} item={item} showSentiment={config.showSentiment} />
      ))}
    </div>
  );
}

function FeedItem({ item, showSentiment }: { item: any; showSentiment?: boolean }) {
  return (
    <div className={`feed-item ${item.sentiment}`}>
      <div className="feed-item-header">
        <span className="author">{item.author?.name || item.source}</span>
        <span className="time">{formatRelativeTime(item.timestamp)}</span>
        {showSentiment && (
          <span className={`sentiment-badge ${item.sentiment}`}>
            {item.sentiment === 'positive' ? '🟢' : item.sentiment === 'negative' ? '🔴' : '⚪'}
            {item.sentimentScore?.toFixed(2)}
          </span>
        )}
      </div>
      <div className="feed-item-body">{item.text || item.title}</div>
      {item.metrics && (
        <div className="feed-item-metrics">
          {item.metrics.likes && <span>❤️ {formatNumber(item.metrics.likes)}</span>}
          {item.metrics.retweets && <span>🔄 {formatNumber(item.metrics.retweets)}</span>}
        </div>
      )}
    </div>
  );
}
```

### MetricCard Panel

```tsx
function MetricCardPanel({ panel }: { panel: PanelRenderData }) {
  const { value, label, unit, change, changePercent, direction, sparkline, subtitle } = panel.data;
  
  return (
    <div className="metric-card">
      <div className="metric-label">{label}</div>
      <div className="metric-value">
        {formatNumber(value)} <span className="metric-unit">{unit}</span>
      </div>
      <div className={`metric-change ${direction}`}>
        {direction === 'up' ? '▲' : '▼'} {formatNumber(Math.abs(change))} ({(changePercent * 100).toFixed(2)}%)
      </div>
      {sparkline && (
        <div className="metric-sparkline">
          <Sparkline data={sparkline} color={direction === 'up' ? '#26a69a' : '#ef5350'} />
        </div>
      )}
      {subtitle && <div className="metric-subtitle">{subtitle}</div>}
    </div>
  );
}
```

## GridStack 布局

```tsx
import { GridStack } from 'gridstack';
import 'gridstack/dist/gridstack.min.css';

function GridLayout({ panels, layout }: { panels: PanelRenderData[]; layout: LayoutConfig }) {
  const gridRef = useRef<HTMLDivElement>(null);
  const gsRef = useRef<GridStack | null>(null);
  
  useEffect(() => {
    if (!gridRef.current) return;
    
    gsRef.current = GridStack.init({
      column: layout.columns,
      cellHeight: 300,
      margin: layout.gap || 8,
      animate: true,
      float: layout.mode === 'freeform',
    }, gridRef.current);
    
    return () => gsRef.current?.destroy(false);
  }, []);
  
  return (
    <div ref={gridRef} className="grid-stack">
      {panels.map(panel => (
        <div
          key={panel.id}
          className="grid-stack-item"
          gs-x={panel.position?.col || 0}
          gs-y={panel.position?.row || 0}
          gs-w={panel.position?.colSpan || 1}
          gs-h={panel.position?.rowSpan || 1}
        >
          <div className="grid-stack-item-content">
            <PanelWrapper panel={panel} />
          </div>
        </div>
      ))}
    </div>
  );
}
```

## Agent 状态反馈

```tsx
function AgentStatus({ status }: { status: AgentStatusMessage | null }) {
  if (!status) return null;
  
  return (
    <div className="agent-status">
      <Spinner size="small" />
      <span>
        {status.type === 'thinking' && '思考中...'}
        {status.type === 'searching' && '搜索数据源...'}
        {status.type === 'executing' && '生成看板...'}
        {status.type === 'mutating' && '更新看板...'}
        {status.type === 'validating' && '验证数据获取代码...'}
      </span>
      {status.detail && <span className="status-detail">{status.detail}</span>}
    </div>
  );
}
```

## 主题

```css
:root {
  /* Dark theme (default) */
  --bg-primary: #0d1117;
  --bg-secondary: #161b22;
  --bg-panel: #1a1f2e;
  --border: #30363d;
  --text-primary: #e6edf3;
  --text-secondary: #8b949e;
  --accent: #58a6ff;
  --positive: #26a69a;
  --negative: #ef5350;
}
```
