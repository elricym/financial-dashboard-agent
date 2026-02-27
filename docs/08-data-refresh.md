# 08 - 数据刷新与缓存

## 核心概念

**刷新 = 重新执行 panel.code。** 每个 panel 的 `code` 字段包含完整的数据获取逻辑，刷新时只需重新执行 code，无需 Agent 参与，无需单独存储 dataSource 参数。

## 刷新机制

### 三种刷新方式

```
1. 手动刷新 — 用户点击 panel 上的刷新按钮 → 后端重新执行 panel.code
2. 自动刷新 — panel 配置了 refreshInterval → 后端定时重新执行 panel.code
3. 对话刷新 — 用户说"更新一下数据" → Agent 调用 mutate 修改 code（或后端直接重执行）
```

### 手动刷新

前端直接调用 API，不经过 Agent。后端重新执行 panel.code：

```typescript
// 前端
async function refreshPanel(dashboardId: string, panelId: string) {
  const res = await fetch(`/api/dashboard/${dashboardId}/panel/${panelId}/refresh`, {
    method: 'POST',
  });
  // 新数据通过 WebSocket 推送
}

// 刷新整个看板的所有 panel
async function refreshDashboard(dashboardId: string) {
  const res = await fetch(`/api/dashboard/${dashboardId}/refresh`, {
    method: 'POST',
  });
}
```

```go
// 后端
func (h *Handler) RefreshPanel(w http.ResponseWriter, r *http.Request) {
    dashboardID := chi.URLParam(r, "dashboardId")
    panelID := chi.URLParam(r, "panelId")
    userID := getUserID(r)
    
    state, _ := h.stateStore.Get(r.Context(), userID, dashboardID)
    panel := state.FindPanel(panelID)
    
    if panel.Code == "" {
        http.Error(w, "panel has no code", 400)
        return
    }
    
    // 在沙箱中重新执行 panel.code
    sandbox := h.sandboxPool.Get()
    defer h.sandboxPool.Put(sandbox)
    sandbox.Inject("sdk", h.cachedAdapters)
    sandbox.Inject("utils", standardUtils)
    
    newData, err := sandbox.Execute(r.Context(), panel.Code)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    
    // 更新 lastRefresh 时间
    panel.LastRefresh = timePtr(time.Now())
    h.stateStore.Save(r.Context(), userID, state)
    
    // 推送新数据到前端
    h.wsHub.Send(state.ID, WSMessage{
        Type: "panel:refreshed",
        Payload: map[string]any{
            "dashboardId": dashboardID,
            "panelId":     panelID,
            "data":        newData,
        },
    })
}

// RefreshDashboard 刷新整个看板
func (h *Handler) RefreshDashboard(w http.ResponseWriter, r *http.Request) {
    dashboardID := chi.URLParam(r, "dashboardId")
    userID := getUserID(r)
    
    state, _ := h.stateStore.Get(r.Context(), userID, dashboardID)
    
    // 并行执行所有 panel.code
    renderData, err := h.codeExecutor.ExecuteAllPanels(r.Context(), state)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    
    // 推送所有 panel 的新数据
    for _, pd := range renderData {
        h.wsHub.Send(state.ID, WSMessage{
            Type: "panel:refreshed",
            Payload: map[string]any{
                "dashboardId": dashboardID,
                "panelId":     pd.ID,
                "data":        pd.Data,
            },
        })
    }
}
```

### 自动刷新

```go
type RefreshScheduler struct {
    stateStore  *FileStateStore
    codeExec    *CodeExecutor
    wsHub       *WSHub
    jobs        map[string]*time.Ticker // key: userID:dashboardID:panelID
    mu          sync.Mutex
}

func (s *RefreshScheduler) SchedulePanel(userID, dashboardID string, panel *Panel) {
    if panel.RefreshInterval == "" {
        return
    }
    
    interval, err := time.ParseDuration(panel.RefreshInterval)
    if err != nil || interval < 10*time.Second {
        return // 最小 10 秒
    }
    
    key := fmt.Sprintf("%s:%s:%s", userID, dashboardID, panel.ID)
    
    s.mu.Lock()
    defer s.mu.Unlock()
    
    if old, exists := s.jobs[key]; exists {
        old.Stop()
    }
    
    ticker := time.NewTicker(interval)
    s.jobs[key] = ticker
    
    go func() {
        for range ticker.C {
            s.refreshPanel(context.Background(), userID, dashboardID, panel.ID)
        }
    }()
}

func (s *RefreshScheduler) refreshPanel(ctx context.Context, userID, dashboardID, panelID string) {
    state, err := s.stateStore.Get(ctx, userID, dashboardID)
    if err != nil {
        return
    }
    
    panel := state.FindPanel(panelID)
    if panel == nil || panel.Code == "" {
        return
    }
    
    // 重新执行 panel.code
    sandbox := s.codeExec.sandboxPool.Get()
    defer s.codeExec.sandboxPool.Put(sandbox)
    sandbox.Inject("sdk", s.codeExec.adapters)
    sandbox.Inject("utils", standardUtils)
    
    newData, err := sandbox.Execute(ctx, panel.Code)
    if err != nil {
        return // 静默失败，下次重试
    }
    
    panel.LastRefresh = timePtr(time.Now())
    s.stateStore.Save(ctx, userID, state)
    
    s.wsHub.Send(dashboardID, WSMessage{
        Type: "panel:refreshed",
        Payload: map[string]any{
            "dashboardId": dashboardID,
            "panelId":     panelID,
            "data":        newData,
        },
    })
}
```

## 为什么不需要单独的 dataSource 参数？

旧设计中 `dataSource` 存储了 `{ source, method, params }`，刷新时通过这些参数重新调用 SDK。

新设计中，`code` 字段本身就是完整的数据获取逻辑：

```javascript
// panel.code 包含了所有信息
"const data = await sdk.market.getKline({ symbol: 'XAUUSD', timeframe: '1D', limit: 120 }); return data;"
```

- **code 是自包含的** — 不需要从外部传入参数
- **code 可以包含复杂逻辑** — 多数据源组合、计算、转换
- **dataSource 保留为可选元数据** — 仅用于 UI 展示"数据来源: market.getKline"

## 缓存策略

### 多层缓存

```
panel.code 执行 → SDK 调用 → L1 内存缓存 → L2 Redis 缓存 → 实际外部 API
```

缓存发生在 SDK adapter 层，对 panel.code 透明：

```go
type CachedAdapterRegistry struct {
    adapters  *AdapterRegistry
    l1Cache   *lru.Cache[string, cacheEntry]
    redis     *redis.Client
    specStore *Registry
}

type cacheEntry struct {
    Data      interface{}
    ExpiresAt time.Time
}

func (c *CachedAdapterRegistry) Call(ctx context.Context, source, method string, params map[string]interface{}) (interface{}, error) {
    cacheKey := c.buildCacheKey(source, method, params)
    ttl := c.getTTL(source, method)
    
    // L1: 内存
    if entry, ok := c.l1Cache.Get(cacheKey); ok {
        if time.Now().Before(entry.ExpiresAt) {
            return entry.Data, nil
        }
        c.l1Cache.Remove(cacheKey)
    }
    
    // L2: Redis
    if ttl > 0 {
        cached, err := c.redis.Get(ctx, "cache:"+cacheKey).Bytes()
        if err == nil {
            var data interface{}
            json.Unmarshal(cached, &data)
            c.l1Cache.Add(cacheKey, cacheEntry{Data: data, ExpiresAt: time.Now().Add(ttl)})
            return data, nil
        }
    }
    
    // 实际调用
    data, err := c.adapters.Call(ctx, source, method, params)
    if err != nil {
        return nil, err
    }
    
    // 写缓存
    if ttl > 0 {
        c.l1Cache.Add(cacheKey, cacheEntry{Data: data, ExpiresAt: time.Now().Add(ttl)})
        encoded, _ := json.Marshal(data)
        c.redis.Set(ctx, "cache:"+cacheKey, encoded, ttl)
    }
    
    return data, nil
}

func (c *CachedAdapterRegistry) buildCacheKey(source, method string, params map[string]interface{}) string {
    keys := make([]string, 0, len(params))
    for k := range params {
        keys = append(keys, k)
    }
    sort.Strings(keys)
    
    var buf strings.Builder
    buf.WriteString(source + ":" + method)
    for _, k := range keys {
        buf.WriteString(":" + k + "=" + fmt.Sprint(params[k]))
    }
    return buf.String()
}

func (c *CachedAdapterRegistry) getTTL(source, method string) time.Duration {
    spec, ok := c.specStore.sources[source]
    if !ok {
        return 0
    }
    for _, m := range spec.Methods {
        if m.Name == method && m.CacheTTL != "" {
            d, _ := time.ParseDuration(m.CacheTTL)
            return d
        }
    }
    return time.Minute
}
```

### 缓存 TTL 指导

| 数据类型 | 建议 TTL | 理由 |
|---|---|---|
| 实时报价 | 5-10s | 价格快速变化 |
| K线数据 | 1-5m | 最新一根K线在变 |
| 日线收盘 | 5-30m | 盘后不变，盘中最后一天在变 |
| 财报数据 | 1h-24h | 更新频率低 |
| COT持仓 | 6h | 每周更新一次 |
| 宏观指标 | 1h | 更新频率低 |
| 新闻 | 5-15m | 需要一定时效性 |
| 推文 | 5-10m | 需要时效性 |
| 链上数据 | 5-15m | 区块确认时间 |
| 经济日历 | 1h | 日程提前已知 |

### 请求去重

同一时刻多个 panel.code 可能调用相同 SDK 方法（如多个 panel 都查 XAUUSD 日线）：

```go
type RequestDeduper struct {
    inflight sync.Map // key → *call
}

type call struct {
    wg  sync.WaitGroup
    val interface{}
    err error
}

func (d *RequestDeduper) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
    if c, loaded := d.inflight.LoadOrStore(key, &call{}); loaded {
        existing := c.(*call)
        existing.wg.Wait()
        return existing.val, existing.err
    }
    
    c, _ := d.inflight.Load(key)
    entry := c.(*call)
    entry.wg.Add(1)
    
    entry.val, entry.err = fn()
    entry.wg.Done()
    
    d.inflight.Delete(key)
    return entry.val, entry.err
}
```

## 限流

### Per-Source 限流

```go
type RateLimiter struct {
    limiters map[string]*rate.Limiter
}

func NewRateLimiter(registry *Registry) *RateLimiter {
    rl := &RateLimiter{limiters: make(map[string]*rate.Limiter)}
    for name, spec := range registry.sources {
        if spec.RateLimit != nil {
            rps := float64(spec.RateLimit.RequestsPerMinute) / 60.0
            rl.limiters[name] = rate.NewLimiter(rate.Limit(rps), spec.RateLimit.RequestsPerMinute/6)
        }
    }
    return rl
}

func (rl *RateLimiter) Wait(ctx context.Context, source string) error {
    if l, ok := rl.limiters[source]; ok {
        return l.Wait(ctx)
    }
    return nil
}
```
