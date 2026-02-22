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
            ├── PanelHeader    // 标题、操作按钮
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
  | { type: 'dashboard:created'; payload: { dashboardId: string; state: DashboardState } }
  | { type: 'dashboard:updated'; payload: { dashboardId: string; diff: StateDiff } }
  | { type: 'dashboard:deleted'; payload: { dashboardId: string } }
  | { type: 'agent:thinking';    payload: { message: string } }        // Agent 正在思考
  | { type: 'agent:executing';   payload: { tool: string; code?: string } } // Agent 正在执行 tool
  | { type: 'agent:message';     payload: { text: string } }           // Agent 文字回复
  | { type: 'panel:refreshed';   payload: { dashboardId: string; panelId: string; data: any } }
  | { type: 'error';             payload: { message: string; code?: string } }
```

### 前端状态更新

```typescript
// 使用 zustand 或类似状态管理
interface DashboardStore {
  dashboards: Map<string, DashboardState>;
  activeDashboardId: string | null;
  
  // Actions
  applyDiff(dashboardId: string, diff: StateDiff): void;
  setActive(dashboardId: string): void;
}

function applyDiff(state: DashboardState, diff: StateDiff): DashboardState {
  const newState = structuredClone(state);
  
  // 删除 panel
  if (diff.removedPanels?.length) {
    newState.panels = newState.panels.filter(p => !diff.removedPanels.includes(p.id));
  }
  
  // 新增 panel
  if (diff.addedPanels?.length) {
    newState.panels.push(...diff.addedPanels);
  }
  
  // 更新 panel
  if (diff.updatedPanels?.length) {
    for (const update of diff.updatedPanels) {
      const idx = newState.panels.findIndex(p => p.id === update.panelId);
      if (idx !== -1) {
        newState.panels[idx] = update.panel;
      }
    }
  }
  
  // 布局变更
  if (diff.layoutChanged) {
    newState.layout = diff.newLayout!;
  }
  
  return newState;
}
```

## 渲染器实现

### PanelWrapper 分发

```tsx
function PanelContent({ panel }: { panel: Panel }) {
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

function CustomPanelRouter({ panel }: { panel: Panel }) {
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

function LWChartPanel({ panel }: { panel: Panel }) {
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
      crosshair: { mode: 0 }, // Normal
    });
    
    // 根据 panel type 创建对应 series
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
    
    // 添加技术指标
    if (panel.indicators) {
      for (const ind of panel.indicators) {
        addIndicatorToChart(chart, series, panel.data, ind);
      }
    }
    
    chart.timeScale().fitContent();
    chartRef.current = chart;
    seriesRef.current = series;
    
    // Resize observer
    const ro = new ResizeObserver(() => {
      chart.applyOptions({
        width: containerRef.current!.clientWidth,
        height: containerRef.current!.clientHeight,
      });
    });
    ro.observe(containerRef.current);
    
    return () => { ro.disconnect(); chart.remove(); };
  }, []);
  
  // 数据更新
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
      
      // Upper band
      chart.addLineSeries({ color: 'rgba(255,152,0,0.5)', lineWidth: 1, priceLineVisible: false })
        .setData(upper.map((v, i) => ({ time: times[i], value: v })).filter(d => d.value !== null));
      // Middle
      chart.addLineSeries({ color: 'rgba(255,152,0,0.8)', lineWidth: 1, priceLineVisible: false })
        .setData(middle.map((v, i) => ({ time: times[i], value: v })).filter(d => d.value !== null));
      // Lower band  
      chart.addLineSeries({ color: 'rgba(255,152,0,0.5)', lineWidth: 1, priceLineVisible: false })
        .setData(lower.map((v, i) => ({ time: times[i], value: v })).filter(d => d.value !== null));
      break;
    }
    
    case 'RSI': {
      // RSI 需要独立的 price scale
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
      // 配置 RSI 刻度
      chart.priceScale('rsi').applyOptions({
        scaleMargins: { top: 0.8, bottom: 0 },
      });
      break;
    }
    
    // MACD, KDJ 等类似...
  }
}
```

### ECharts Panel

```tsx
import * as echarts from 'echarts';

function EChartsPanel({ panel }: { panel: Panel }) {
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

function buildEChartsOption(panel: Panel): echarts.EChartsOption {
  switch (panel.type) {
    case 'heatmap':
      return buildHeatmapOption(panel);
    case 'pie':
      return buildPieOption(panel);
    case 'radar':
      return buildRadarOption(panel);
    case 'bar':
      return buildBarOption(panel);
    case 'scatter':
      return buildScatterOption(panel);
    case 'treemap':
      return buildTreemapOption(panel);
    case 'sankey':
      return buildSankeyOption(panel);
    default:
      return {};
  }
}

function buildHeatmapOption(panel: Panel): echarts.EChartsOption {
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

// ... buildPieOption, buildBarOption 等类似
```

### Feed Panel

```tsx
function FeedPanel({ panel }: { panel: Panel }) {
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
  
  const grouped = useMemo(() => {
    if (!config.groupBy) return null;
    const groups: Record<string, any[]> = {};
    for (const item of sortedItems) {
      const key = item[config.groupBy] || 'unknown';
      (groups[key] ||= []).push(item);
    }
    return groups;
  }, [sortedItems, config.groupBy]);
  
  if (grouped) {
    return (
      <div className="feed-panel grouped">
        {Object.entries(grouped).map(([group, items]) => (
          <div key={group} className="feed-group">
            <h4 className="feed-group-title">{group}</h4>
            {items.map(item => <FeedItem key={item.id} item={item} showSentiment={config.showSentiment} />)}
          </div>
        ))}
      </div>
    );
  }
  
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
function MetricCardPanel({ panel }: { panel: Panel }) {
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

function GridLayout({ state }: { state: DashboardState }) {
  const gridRef = useRef<HTMLDivElement>(null);
  const gsRef = useRef<GridStack | null>(null);
  
  useEffect(() => {
    if (!gridRef.current) return;
    
    gsRef.current = GridStack.init({
      column: state.layout.columns,
      cellHeight: 300,
      margin: state.layout.gap || 8,
      animate: true,
      float: state.layout.mode === 'freeform',
    }, gridRef.current);
    
    return () => gsRef.current?.destroy(false);
  }, []);
  
  return (
    <div ref={gridRef} className="grid-stack">
      {state.panels.map(panel => (
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

用户发送消息后，前端展示 Agent 的工作状态：

```tsx
function AgentStatus({ status }: { status: AgentStatusMessage | null }) {
  if (!status) return null;
  
  return (
    <div className="agent-status">
      <Spinner size="small" />
      <span>
        {status.type === 'thinking' && '思考中...'}
        {status.type === 'searching' && '搜索数据源...'}
        {status.type === 'executing' && '获取数据并生成看板...'}
        {status.type === 'mutating' && '更新看板...'}
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
