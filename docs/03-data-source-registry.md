# 03 - 数据源注册表

## 概述

数据源注册表是 Agent 发现和调用外部数据的中枢。每个数据 SDK 通过注册 spec 接入系统，Agent 在运行时通过 `search` tool 搜索 spec 来发现所需数据源。

## Spec 结构定义

```go
// DataSourceSpec 数据源完整规格
type DataSourceSpec struct {
    Name        string       `json:"name" yaml:"name"`               // 唯一标识符
    DisplayName string       `json:"displayName" yaml:"displayName"` // 中文展示名
    Description string       `json:"description" yaml:"description"` // 一句话描述能力
    Tags        []string     `json:"tags" yaml:"tags"`               // 分类标签
    Methods     []MethodSpec `json:"methods" yaml:"methods"`         // 方法列表
    RateLimit   *RateLimit   `json:"rateLimit,omitempty" yaml:"rateLimit,omitempty"`
}

// MethodSpec 方法规格
type MethodSpec struct {
    Name        string      `json:"name" yaml:"name"`
    Description string      `json:"description" yaml:"description"`
    Params      []ParamSpec `json:"params" yaml:"params"`
    Returns     ReturnSpec  `json:"returns" yaml:"returns"`
    Example     string      `json:"example,omitempty" yaml:"example,omitempty"` // 调用示例
    CacheTTL    string      `json:"cacheTTL,omitempty" yaml:"cacheTTL,omitempty"` // 建议缓存时间
}

// ParamSpec 参数规格
type ParamSpec struct {
    Name        string      `json:"name" yaml:"name"`
    Type        string      `json:"type" yaml:"type"`           // string, number, boolean, string[], object
    Required    bool        `json:"required" yaml:"required"`
    Description string      `json:"description" yaml:"description"`
    Enum        []string    `json:"enum,omitempty" yaml:"enum,omitempty"`       // 可选值列表
    Default     interface{} `json:"default,omitempty" yaml:"default,omitempty"` // 默认值
    Min         *float64    `json:"min,omitempty" yaml:"min,omitempty"`
    Max         *float64    `json:"max,omitempty" yaml:"max,omitempty"`
}

// ReturnSpec 返回值规格
type ReturnSpec struct {
    Type        string `json:"type" yaml:"type"`               // 类型标识
    Description string `json:"description" yaml:"description"` // 返回值描述
    Schema      string `json:"schema,omitempty" yaml:"schema,omitempty"` // JSON Schema（可选）
}

// RateLimit 限流配置
type RateLimit struct {
    RequestsPerMinute int `json:"requestsPerMinute" yaml:"requestsPerMinute"`
    RequestsPerDay    int `json:"requestsPerDay,omitempty" yaml:"requestsPerDay,omitempty"`
}
```

## Spec 文件示例

数据源 spec 以 YAML 文件维护，每个文件一个数据源：

### `specs/market.yaml`

```yaml
name: market
displayName: 行情数据
description: 股票、外汇、加密货币、商品的实时和历史行情数据（OHLCV、报价、深度）
tags: [行情, 价格, K线, OHLCV, 报价, 深度, 实时]
rateLimit:
  requestsPerMinute: 300
methods:
  - name: getKline
    description: 获取K线(OHLCV)数据
    params:
      - { name: symbol, type: string, required: true, description: "交易对代码，如 XAUUSD, BTCUSD, AAPL, 600519.SH" }
      - { name: timeframe, type: string, required: true, description: "K线周期", enum: ["1m","5m","15m","30m","1H","4H","1D","1W","1M"] }
      - { name: limit, type: number, required: false, description: "返回条数", default: 100, max: 1000 }
      - { name: startTime, type: string, required: false, description: "起始时间 ISO8601" }
      - { name: endTime, type: string, required: false, description: "结束时间 ISO8601" }
    returns:
      type: "OHLCV[]"
      description: "K线数组，每条包含 timestamp, open, high, low, close, volume"
    example: "sdk.market.getKline({ symbol: 'XAUUSD', timeframe: '1D', limit: 120 })"
    cacheTTL: "1m"

  - name: getQuote
    description: 获取最新报价（实时价格、涨跌幅）
    params:
      - { name: symbols, type: "string[]", required: true, description: "交易对列表" }
    returns:
      type: "Quote[]"
      description: "报价数组，含 symbol, price, change, changePercent, volume, timestamp"
    cacheTTL: "10s"

  - name: getDepth
    description: 获取订单簿深度
    params:
      - { name: symbol, type: string, required: true }
      - { name: limit, type: number, required: false, default: 20 }
    returns:
      type: "OrderBook"
      description: "买卖盘口，含 bids[], asks[]，每档 [price, quantity]"
    cacheTTL: "5s"

  - name: getDaily
    description: 获取日线收盘价序列（轻量，适合计算指标和相关性）
    params:
      - { name: symbol, type: string, required: true }
      - { name: limit, type: number, required: false, default: 252 }
    returns:
      type: "DailyPrice[]"
      description: "日线数组，含 date, close, volume"
    cacheTTL: "5m"
```

### `specs/social.yaml`

```yaml
name: social
displayName: 社交媒体数据
description: Twitter/X、Reddit、StockTwits 等平台的金融相关社媒数据和情绪分析
tags: [舆情, 社交, Twitter, Reddit, 情绪, 热度]
rateLimit:
  requestsPerMinute: 60
methods:
  - name: searchTweets
    description: 搜索金融相关推文
    params:
      - { name: query, type: string, required: true, description: "搜索关键词或 cashtag（如 $GOLD, $BTC）" }
      - { name: limit, type: number, required: false, default: 50, max: 200 }
      - { name: sentiment, type: boolean, required: false, default: true, description: "是否附带情绪分析" }
      - { name: lang, type: string, required: false, default: "en" }
      - { name: since, type: string, required: false, description: "起始时间 ISO8601" }
    returns:
      type: "Tweet[]"
      description: "推文数组，含 id, text, author, timestamp, likes, retweets, sentiment(positive/negative/neutral), sentimentScore"
    cacheTTL: "5m"

  - name: getTrending
    description: 获取金融话题热度排行
    params:
      - { name: category, type: string, required: false, description: "分类筛选", enum: ["stocks","crypto","forex","commodities","macro","all"] }
      - { name: limit, type: number, required: false, default: 20 }
    returns:
      type: "TrendingTopic[]"
      description: "热门话题数组，含 topic, mentionCount, sentimentRatio, momentum"
    cacheTTL: "10m"

  - name: getSentimentTimeline
    description: 获取某个话题的情绪随时间变化
    params:
      - { name: query, type: string, required: true }
      - { name: interval, type: string, required: false, default: "1H", enum: ["1H","4H","1D"] }
      - { name: limit, type: number, required: false, default: 168, description: "数据点数" }
    returns:
      type: "SentimentPoint[]"
      description: "时间序列，含 timestamp, positive, negative, neutral, total, score"
    cacheTTL: "15m"
```

### `specs/fundamental.yaml`

```yaml
name: fundamental
displayName: 基本面数据
description: 公司财报、估值指标、财务比率、盈利预期
tags: [财报, 基本面, 估值, 盈利, 财务]
methods:
  - name: getFinancials
    description: 获取公司财务报表数据
    params:
      - { name: symbol, type: string, required: true }
      - { name: statement, type: string, required: true, enum: ["income","balance","cashflow"] }
      - { name: period, type: string, required: false, default: "quarterly", enum: ["quarterly","annual"] }
      - { name: limit, type: number, required: false, default: 8 }
    returns:
      type: "FinancialStatement[]"
      description: "财务报表数组"

  - name: getValuation
    description: 获取估值指标
    params:
      - { name: symbol, type: string, required: true }
    returns:
      type: "Valuation"
      description: "估值指标，含 PE, PB, PS, EV/EBITDA, marketCap, dividendYield"

  - name: getEarnings
    description: 获取盈利数据和分析师预期
    params:
      - { name: symbol, type: string, required: true }
      - { name: limit, type: number, required: false, default: 8 }
    returns:
      type: "EarningsData[]"
      description: "盈利数据，含 date, actualEPS, estimatedEPS, surprise, revenueActual, revenueEstimated"
```

### `specs/macro.yaml`

```yaml
name: macro
displayName: 宏观经济数据
description: GDP、CPI、利率、就业、PMI 等宏观经济指标和央行数据
tags: [宏观, 经济, GDP, CPI, 利率, 就业, 央行]
methods:
  - name: getIndicator
    description: 获取宏观经济指标时间序列
    params:
      - { name: indicator, type: string, required: true, description: "指标代码", enum: ["GDP","CPI","PPI","UNEMPLOYMENT","PMI","RETAIL_SALES","INDUSTRIAL_PRODUCTION","HOUSING_STARTS"] }
      - { name: country, type: string, required: false, default: "US", enum: ["US","CN","EU","JP","UK","DE"] }
      - { name: limit, type: number, required: false, default: 40 }
    returns:
      type: "MacroPoint[]"
      description: "时间序列，含 date, value, previousValue, forecast"

  - name: getInterestRates
    description: 获取央行利率和国债收益率
    params:
      - { name: country, type: string, required: false, default: "US" }
      - { name: types, type: "string[]", required: false, description: "利率类型", enum: ["federal_funds","10Y","2Y","30Y","spread_10Y_2Y"] }
    returns:
      type: "RateData"
      description: "利率数据，含各类型当前值和历史序列"

  - name: getEconomicCalendar
    description: 获取经济日历（即将发布的经济数据）
    params:
      - { name: days, type: number, required: false, default: 7, description: "未来天数" }
      - { name: importance, type: string, required: false, enum: ["high","medium","low","all"], default: "high" }
      - { name: country, type: string, required: false }
    returns:
      type: "CalendarEvent[]"
      description: "经济事件数组，含 datetime, country, event, importance, forecast, previous"
```

### `specs/onchain.yaml`

```yaml
name: onchain
displayName: 链上数据
description: 区块链链上指标、DeFi 数据、鲸鱼活动、Gas 费用
tags: [链上, 区块链, DeFi, 鲸鱼, Gas, TVL]
methods:
  - name: getMetrics
    description: 获取链上指标
    params:
      - { name: chain, type: string, required: true, enum: ["bitcoin","ethereum","solana","base"] }
      - { name: metrics, type: "string[]", required: true, description: "指标列表", enum: ["active_addresses","transaction_count","hash_rate","difficulty","nvt","mvrv","sopr"] }
      - { name: limit, type: number, required: false, default: 90 }
    returns:
      type: "ChainMetric[]"
      description: "链上指标时间序列"

  - name: getWhaleActivity
    description: 获取大户/鲸鱼链上活动
    params:
      - { name: token, type: string, required: true, description: "代币，如 BTC, ETH" }
      - { name: minUSD, type: number, required: false, default: 1000000, description: "最小金额(USD)" }
      - { name: limit, type: number, required: false, default: 50 }
    returns:
      type: "WhaleTransaction[]"
      description: "大额交易数组，含 hash, from, to, amount, usdValue, timestamp, type(transfer/exchange_deposit/exchange_withdrawal)"

  - name: getDeFiTVL
    description: 获取 DeFi 协议 TVL 数据
    params:
      - { name: protocol, type: string, required: false, description: "协议名，不指定返回总览" }
      - { name: chain, type: string, required: false }
      - { name: limit, type: number, required: false, default: 90 }
    returns:
      type: "TVLData[]"
      description: "TVL 时间序列"
```

### `specs/news.yaml`

```yaml
name: news
displayName: 新闻资讯
description: 金融新闻、研报摘要、公告、事件
tags: [新闻, 研报, 公告, 事件, 快讯]
methods:
  - name: getNews
    description: 获取金融新闻
    params:
      - { name: query, type: string, required: false, description: "搜索关键词" }
      - { name: category, type: string, required: false, enum: ["market","economy","company","crypto","commodity","forex"] }
      - { name: limit, type: number, required: false, default: 30 }
      - { name: lang, type: string, required: false, default: "zh" }
    returns:
      type: "NewsItem[]"
      description: "新闻数组，含 title, summary, source, url, publishedAt, tags, sentiment"

  - name: getResearchNotes
    description: 获取机构研报摘要
    params:
      - { name: symbol, type: string, required: false }
      - { name: sector, type: string, required: false }
      - { name: limit, type: number, required: false, default: 20 }
    returns:
      type: "ResearchNote[]"
      description: "研报摘要，含 title, institution, analyst, rating, targetPrice, summary, publishedAt"
```

### `specs/cot.yaml`

```yaml
name: cot
displayName: 持仓数据
description: CFTC COT 报告、期货持仓、ETF 持仓流入流出
tags: [持仓, COT, 期货, ETF, 资金流向]
methods:
  - name: getCOTReport
    description: 获取 CFTC COT 持仓报告
    params:
      - { name: symbol, type: string, required: true, description: "品种，如 GOLD, SILVER, CRUDE_OIL, SP500, EURUSD" }
      - { name: limit, type: number, required: false, default: 52 }
    returns:
      type: "COTData[]"
      description: "COT 数据，含 date, commercialLong, commercialShort, nonCommercialLong, nonCommercialShort, openInterest"

  - name: getETFFlows
    description: 获取 ETF 资金流入流出
    params:
      - { name: symbol, type: string, required: true, description: "ETF 代码，如 GLD, SLV, SPY, QQQ" }
      - { name: limit, type: number, required: false, default: 60 }
    returns:
      type: "ETFFlow[]"
      description: "资金流数据，含 date, inflow, outflow, netFlow, aum, shares"
```

## 注册表加载

```go
type Registry struct {
    sources map[string]*DataSourceSpec
    mu      sync.RWMutex
}

func NewRegistry(specDir string) (*Registry, error) {
    r := &Registry{sources: make(map[string]*DataSourceSpec)}
    
    files, _ := filepath.Glob(filepath.Join(specDir, "*.yaml"))
    for _, f := range files {
        var spec DataSourceSpec
        data, _ := os.ReadFile(f)
        yaml.Unmarshal(data, &spec)
        r.sources[spec.Name] = &spec
    }
    return r, nil
}

// ToJSObject 将注册表转为 JS 可访问的对象
func (r *Registry) ToJSObject() map[string]interface{} {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    sources := make([]interface{}, 0, len(r.sources))
    for _, spec := range r.sources {
        sources = append(sources, specToMap(spec))
    }
    return map[string]interface{}{
        "sources": sources,
    }
}
```

## SDK Adapter 接口

每个数据源需要实现统一的 Adapter 接口：

```go
// Adapter 数据源适配器
type Adapter interface {
    // Name 返回数据源名称，需与 spec 中的 name 一致
    Name() string
    
    // Call 调用方法
    Call(ctx context.Context, method string, params map[string]interface{}) (interface{}, error)
    
    // HealthCheck 健康检查
    HealthCheck(ctx context.Context) error
}

// AdapterRegistry 管理所有适配器
type AdapterRegistry struct {
    adapters map[string]Adapter
}

func (ar *AdapterRegistry) Register(adapter Adapter) {
    ar.adapters[adapter.Name()] = adapter
}

func (ar *AdapterRegistry) Call(ctx context.Context, source, method string, params map[string]interface{}) (interface{}, error) {
    adapter, ok := ar.adapters[source]
    if !ok {
        return nil, fmt.Errorf("unknown data source: %s", source)
    }
    return adapter.Call(ctx, method, params)
}
```

### 适配器实现示例

```go
type MarketAdapter struct {
    client *marketapi.Client
}

func (a *MarketAdapter) Name() string { return "market" }

func (a *MarketAdapter) Call(ctx context.Context, method string, params map[string]interface{}) (interface{}, error) {
    switch method {
    case "getKline":
        symbol := params["symbol"].(string)
        timeframe := params["timeframe"].(string)
        limit := intParam(params, "limit", 100)
        return a.client.GetKline(ctx, symbol, timeframe, limit)
    case "getQuote":
        symbols := stringSliceParam(params, "symbols")
        return a.client.GetQuote(ctx, symbols)
    case "getDepth":
        symbol := params["symbol"].(string)
        limit := intParam(params, "limit", 20)
        return a.client.GetDepth(ctx, symbol, limit)
    case "getDaily":
        symbol := params["symbol"].(string)
        limit := intParam(params, "limit", 252)
        return a.client.GetDaily(ctx, symbol, limit)
    default:
        return nil, fmt.Errorf("unknown method: %s.%s", a.Name(), method)
    }
}
```

## 参数校验

在 SDK Proxy 层调用 Adapter 之前，根据 spec 自动校验参数：

```go
func ValidateParams(spec *MethodSpec, params map[string]interface{}) error {
    for _, p := range spec.Params {
        val, exists := params[p.Name]
        
        // 必填检查
        if p.Required && !exists {
            return fmt.Errorf("missing required param: %s", p.Name)
        }
        
        // 填充默认值
        if !exists && p.Default != nil {
            params[p.Name] = p.Default
            continue
        }
        
        if !exists {
            continue
        }
        
        // 枚举检查
        if len(p.Enum) > 0 {
            if !contains(p.Enum, fmt.Sprint(val)) {
                return fmt.Errorf("param %s: value %v not in enum %v", p.Name, val, p.Enum)
            }
        }
        
        // 范围检查
        if p.Min != nil || p.Max != nil {
            num, ok := toFloat64(val)
            if ok {
                if p.Min != nil && num < *p.Min {
                    return fmt.Errorf("param %s: %v < min %v", p.Name, val, *p.Min)
                }
                if p.Max != nil && num > *p.Max {
                    return fmt.Errorf("param %s: %v > max %v", p.Name, val, *p.Max)
                }
            }
        }
    }
    return nil
}
```
