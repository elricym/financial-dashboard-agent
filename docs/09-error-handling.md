# 09 - 错误处理与降级

## 错误分类

```
┌─────────────────────────────────────────────────────────────┐
│                     错误类型                                  │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ 沙箱执行错误  │  │ 数据源错误    │  │ Agent 逻辑错误    │  │
│  │              │  │              │  │                  │  │
│  │ - 语法错误    │  │ - API 不可用  │  │ - 无效 State     │  │
│  │ - 运行时异常  │  │ - 限流       │  │ - 缺少必需字段    │  │
│  │ - 超时       │  │ - 认证失败    │  │ - 类型不匹配     │  │
│  │ - 内存溢出    │  │ - 数据格式变更 │  │ - 引用不存在的   │  │
│  │              │  │ - 超时       │  │   panel          │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 错误处理流程

### 沙箱执行错误

```go
type SandboxError struct {
    Type    string `json:"type"`    // "syntax", "runtime", "timeout", "memory"
    Message string `json:"message"`
    Line    int    `json:"line,omitempty"`
    Stack   string `json:"stack,omitempty"`
}

func (mgr *SandboxManager) executeWithRecovery(ctx context.Context, sb *Sandbox, code string) (result interface{}, err error) {
    defer func() {
        if r := recover(); r != nil {
            switch v := r.(type) {
            case *goja.InterruptedError:
                err = &SandboxError{Type: "timeout", Message: "execution timed out (30s limit)"}
            case *goja.Exception:
                err = &SandboxError{Type: "runtime", Message: v.String(), Stack: v.String()}
            default:
                err = &SandboxError{Type: "runtime", Message: fmt.Sprint(v)}
            }
        }
    }()
    
    result, err = mgr.executeWithTimeout(ctx, sb, code)
    if err != nil {
        return nil, &SandboxError{Type: "syntax", Message: err.Error()}
    }
    return result, nil
}
```

### 错误返回给 Agent

所有 tool 执行错误都结构化返回给 Agent，让 Agent 决定如何处理：

```go
func (h *ToolHandler) handleExecute(ctx context.Context, input ExecuteInput) ExecuteOutput {
    state, err := h.sandbox.ExecuteCode(ctx, input.Code)
    if err != nil {
        return ExecuteOutput{
            Error: formatErrorForAgent(err),
        }
    }
    
    // 验证 state
    if validationErr := validateDashboardState(state); validationErr != nil {
        return ExecuteOutput{
            Error: fmt.Sprintf("生成的看板状态无效: %s。请修正代码后重试。", validationErr),
        }
    }
    
    // 成功
    h.stateStore.Save(ctx, state)
    h.wsHub.Push(state)
    return ExecuteOutput{
        DashboardID: state.ID,
        State:       state.ToSummary(),
    }
}

func formatErrorForAgent(err error) string {
    switch e := err.(type) {
    case *SandboxError:
        switch e.Type {
        case "timeout":
            return "代码执行超时(30秒)。可能是数据量过大或存在死循环。请优化代码或减少数据请求量。"
        case "syntax":
            return fmt.Sprintf("代码语法错误: %s", e.Message)
        case "runtime":
            return fmt.Sprintf("代码运行时错误: %s", e.Message)
        case "memory":
            return "内存超限。请减少数据量或避免在内存中构建过大的对象。"
        }
    case *SDKError:
        return fmt.Sprintf("数据源调用失败: %s.%s — %s", e.Source, e.Method, e.Message)
    }
    return err.Error()
}
```

### Agent 自动重试

Agent 收到错误后可能自动修正。系统配置最大重试次数：

```go
type AgentConfig struct {
    MaxToolRetries int // 同一 tool 最大重试次数，建议 2
}
```

Agent 的 system prompt 中指导：
```
如果 tool 返回错误：
- 语法/运行时错误：检查代码，修正后重试（最多重试2次）
- 数据源不可用/限流：告知用户，建议稍后重试或使用替代数据源
- 超时：简化代码或减少数据量后重试
- 如果连续失败2次，停止重试，告知用户具体问题
```

## 数据源降级

### 健康检查

```go
type HealthMonitor struct {
    adapters  *AdapterRegistry
    status    map[string]*SourceStatus
    mu        sync.RWMutex
    interval  time.Duration
}

type SourceStatus struct {
    Name      string    `json:"name"`
    Healthy   bool      `json:"healthy"`
    LastCheck time.Time `json:"lastCheck"`
    LastError string    `json:"lastError,omitempty"`
    Latency   time.Duration `json:"latency"`
    ErrorRate float64   `json:"errorRate"` // 最近 5 分钟错误率
}

func (m *HealthMonitor) Start() {
    ticker := time.NewTicker(m.interval) // 每 30 秒
    go func() {
        for range ticker.C {
            m.checkAll()
        }
    }()
}

func (m *HealthMonitor) checkAll() {
    for name, adapter := range m.adapters.adapters {
        start := time.Now()
        err := adapter.HealthCheck(context.Background())
        latency := time.Since(start)
        
        m.mu.Lock()
        status := m.status[name]
        status.LastCheck = time.Now()
        status.Latency = latency
        if err != nil {
            status.Healthy = false
            status.LastError = err.Error()
        } else {
            status.Healthy = true
            status.LastError = ""
        }
        m.mu.Unlock()
    }
}

// IsHealthy 供 SDK Proxy 调用前检查
func (m *HealthMonitor) IsHealthy(source string) bool {
    m.mu.RLock()
    defer m.mu.RUnlock()
    if s, ok := m.status[source]; ok {
        return s.Healthy
    }
    return true // 未知状态视为健康
}
```

### 降级策略

数据源不可用时的处理：

```go
type SDKProxy struct {
    adapters      *CachedAdapterRegistry
    healthMonitor *HealthMonitor
    rateLimiter   *RateLimiter
    deduper       *RequestDeduper
}

func (p *SDKProxy) Call(ctx context.Context, source, method string, params map[string]interface{}) (interface{}, error) {
    // 1. 健康检查
    if !p.healthMonitor.IsHealthy(source) {
        // 尝试返回过期缓存
        if cached := p.adapters.GetExpiredCache(source, method, params); cached != nil {
            return cached, nil // 返回旧数据，前端显示"数据可能不是最新"
        }
        return nil, &SDKError{
            Source:  source,
            Method:  method,
            Message: fmt.Sprintf("数据源 %s 当前不可用", source),
            Retryable: true,
        }
    }
    
    // 2. 限流
    if err := p.rateLimiter.Wait(ctx, source); err != nil {
        return nil, &SDKError{
            Source:  source,
            Method:  method,
            Message: "请求过于频繁，请稍后重试",
            Retryable: true,
        }
    }
    
    // 3. 去重 + 缓存 + 实际调用
    cacheKey := p.adapters.buildCacheKey(source, method, params)
    return p.deduper.Do(cacheKey, func() (interface{}, error) {
        return p.adapters.Call(ctx, source, method, params)
    })
}
```

## State 验证

```go
func validateDashboardState(state *DashboardState) error {
    if state.Title == "" {
        return fmt.Errorf("title is required")
    }
    
    if state.Layout.Columns < 1 || state.Layout.Columns > 6 {
        return fmt.Errorf("layout.columns must be 1-6, got %d", state.Layout.Columns)
    }
    
    ids := make(map[string]bool)
    for i, panel := range state.Panels {
        if panel.ID == "" {
            return fmt.Errorf("panel[%d] missing id", i)
        }
        if ids[panel.ID] {
            return fmt.Errorf("duplicate panel id: %s", panel.ID)
        }
        ids[panel.ID] = true
        
        if panel.Title == "" {
            return fmt.Errorf("panel[%d] (%s) missing title", i, panel.ID)
        }
        if panel.Type == "" {
            return fmt.Errorf("panel[%d] (%s) missing type", i, panel.ID)
        }
        if panel.Renderer == "" {
            return fmt.Errorf("panel[%d] (%s) missing renderer", i, panel.ID)
        }
        if panel.Data == nil {
            return fmt.Errorf("panel[%d] (%s) missing data", i, panel.ID)
        }
    }
    
    if len(state.Panels) > 20 {
        return fmt.Errorf("too many panels: %d (max 20)", len(state.Panels))
    }
    
    return nil
}
```

## 前端错误展示

```tsx
function PanelError({ error, onRetry }: { error: PanelError; onRetry: () => void }) {
  return (
    <div className="panel-error">
      <div className="error-icon">⚠️</div>
      <div className="error-message">{error.message}</div>
      {error.retryable && (
        <button onClick={onRetry} className="retry-button">重试</button>
      )}
      {error.staleData && (
        <div className="stale-notice">显示的数据可能不是最新</div>
      )}
    </div>
  );
}
```
