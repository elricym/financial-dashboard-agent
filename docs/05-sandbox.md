# 05 - 沙箱执行环境

## 概述

Agent 生成的 JavaScript 代码在隔离的沙箱中执行。沙箱提供受控的运行环境，暴露数据源 SDK 和工具函数，同时限制对系统资源的访问。

## 技术选型

### goja (推荐)

[goja](https://github.com/nicehash/goja) 是纯 Go 实现的 ECMAScript 5.1+ 引擎。

**优点：**
- 纯 Go，无 CGO 依赖，编译部署简单
- 轻量，启动快（微秒级）
- 可以精确控制暴露给 JS 的 API
- 天然内存隔离

**缺点：**
- 不完整支持 ES2020+（async/await 需要包装）
- 性能不如 V8（但对这个场景够用 — 代码量小，计算轻）

**async/await 处理：**
goja 原生不支持 async/await，但可以通过以下方式处理：
1. 将 Agent 代码包装为同步执行（SDK 调用在 Go 侧并行，对 JS 暴露同步接口）
2. 或使用 [goja_nodejs](https://github.com/nicehash/goja_nodejs) 的 event loop 支持

### 备选：V8 binding

使用 [nicehash/nicehash-v8go](https://github.com/nicehash/nicehash-v8go) 或类似库。

**优点：**
- 完整 ES2020+ 支持
- 性能优秀
- V8 isolate 天然安全隔离

**缺点：**
- CGO 依赖，编译复杂
- 内存占用较大
- 部署需要 V8 动态库

### 建议

先用 goja，把 SDK 调用包装为同步接口。如果后续需要更复杂的 JS 特性再切 V8。

## 沙箱架构

```
┌───────────────────────────────────────────────┐
│              Sandbox Instance                  │
│                                               │
│  ┌──────────────────────────────────────────┐ │
│  │           goja Runtime                    │ │
│  │                                          │ │
│  │  全局对象:                                │ │
│  │  ├── sdk.market.getKline(params)         │ │
│  │  ├── sdk.social.searchTweets(params)     │ │
│  │  ├── sdk.*.*(params)                     │ │
│  │  ├── spec.sources[]                      │ │
│  │  ├── utils.computeMA(data, period)       │ │
│  │  ├── utils.*()                           │ │
│  │  └── console.log()                       │ │
│  │                                          │ │
│  │  不可访问:                                │ │
│  │  ├── require / import                    │ │
│  │  ├── process / os                        │ │
│  │  ├── fetch / XMLHttpRequest              │ │
│  │  ├── fs / file                           │ │
│  │  └── eval (可选禁用)                     │ │
│  └──────────────────────────────────────────┘ │
│                                               │
│  资源限制:                                     │
│  ├── 执行超时: 30s                             │
│  ├── 内存上限: 64MB                            │
│  └── SDK 调用次数: 50次/execution              │
└───────────────────────────────────────────────┘
```

## 实现

### Sandbox Manager

```go
type SandboxManager struct {
    registry    *Registry          // 数据源 spec
    adapters    *AdapterRegistry   // SDK 适配器
    stateStore  *StateStore        // 看板状态
    pool        chan *Sandbox       // 沙箱池
    config      SandboxConfig
}

type SandboxConfig struct {
    PoolSize          int           // 预创建沙箱数
    ExecutionTimeout  time.Duration // 单次执行超时
    MemoryLimit       int64         // 内存上限 bytes
    MaxSDKCalls       int           // 单次执行最大 SDK 调用数
}

func NewSandboxManager(cfg SandboxConfig, registry *Registry, adapters *AdapterRegistry, store *StateStore) *SandboxManager {
    mgr := &SandboxManager{
        registry:   registry,
        adapters:   adapters,
        stateStore: store,
        pool:       make(chan *Sandbox, cfg.PoolSize),
        config:     cfg,
    }
    // 预创建沙箱
    for i := 0; i < cfg.PoolSize; i++ {
        mgr.pool <- mgr.createSandbox()
    }
    return mgr
}
```

### Sandbox 实例

```go
type Sandbox struct {
    runtime  *goja.Runtime
    sdkCalls int
    logs     []string
    mu       sync.Mutex
}

func (mgr *SandboxManager) createSandbox() *Sandbox {
    rt := goja.New()
    sb := &Sandbox{runtime: rt}
    
    // 注入 spec（只读）
    rt.Set("spec", mgr.registry.ToJSObject())
    
    // 注入 console
    rt.Set("console", map[string]interface{}{
        "log": func(args ...interface{}) {
            sb.mu.Lock()
            sb.logs = append(sb.logs, fmt.Sprint(args...))
            sb.mu.Unlock()
        },
    })
    
    return sb
}
```

### SDK Proxy 注入

```go
func (mgr *SandboxManager) injectSDK(sb *Sandbox, ctx context.Context) {
    sdkObj := mgr.runtime.NewObject()
    
    // 为每个注册的数据源创建代理对象
    for name, spec := range mgr.registry.sources {
        sourceObj := sb.runtime.NewObject()
        
        for _, method := range spec.Methods {
            methodName := method.Name
            methodSpec := method
            
            sourceObj.Set(methodName, func(call goja.FunctionCall) goja.Value {
                // 检查调用次数限制
                sb.mu.Lock()
                sb.sdkCalls++
                if sb.sdkCalls > mgr.config.MaxSDKCalls {
                    sb.mu.Unlock()
                    panic(sb.runtime.NewGoError(fmt.Errorf("exceeded max SDK calls (%d)", mgr.config.MaxSDKCalls)))
                }
                sb.mu.Unlock()
                
                // 解析参数
                params := make(map[string]interface{})
                if len(call.Arguments) > 0 {
                    obj := call.Arguments[0].Export()
                    if m, ok := obj.(map[string]interface{}); ok {
                        params = m
                    }
                }
                
                // 参数校验
                if err := ValidateParams(&methodSpec, params); err != nil {
                    panic(sb.runtime.NewGoError(err))
                }
                
                // 调用实际 SDK
                result, err := mgr.adapters.Call(ctx, name, methodName, params)
                if err != nil {
                    panic(sb.runtime.NewGoError(err))
                }
                
                val, _ := sb.runtime.ToValue(result)
                return val
            })
        }
        
        sdkObj.Set(name, sourceObj)
    }
    
    sb.runtime.Set("sdk", sdkObj)
}
```

### Utils 工具函数注入

```go
func (mgr *SandboxManager) injectUtils(sb *Sandbox) {
    utils := sb.runtime.NewObject()
    
    // 移动平均
    utils.Set("computeMA", func(call goja.FunctionCall) goja.Value {
        data := toFloat64Slice(call.Arguments[0].Export())
        period := int(call.Arguments[1].ToInteger())
        result := indicators.MA(data, period)
        val, _ := sb.runtime.ToValue(result)
        return val
    })
    
    // 指数移动平均
    utils.Set("computeEMA", func(call goja.FunctionCall) goja.Value {
        data := toFloat64Slice(call.Arguments[0].Export())
        period := int(call.Arguments[1].ToInteger())
        result := indicators.EMA(data, period)
        val, _ := sb.runtime.ToValue(result)
        return val
    })
    
    // 布林带
    utils.Set("computeBOLL", func(call goja.FunctionCall) goja.Value {
        data := toFloat64Slice(call.Arguments[0].Export())
        period := int(call.Arguments[1].ToInteger())
        stdDev := call.Arguments[2].ToFloat()
        upper, mid, lower := indicators.BOLL(data, period, stdDev)
        val, _ := sb.runtime.ToValue(map[string]interface{}{
            "upper": upper, "middle": mid, "lower": lower,
        })
        return val
    })
    
    // RSI
    utils.Set("computeRSI", func(call goja.FunctionCall) goja.Value {
        data := toFloat64Slice(call.Arguments[0].Export())
        period := int(call.Arguments[1].ToInteger())
        result := indicators.RSI(data, period)
        val, _ := sb.runtime.ToValue(result)
        return val
    })
    
    // MACD
    utils.Set("computeMACD", func(call goja.FunctionCall) goja.Value {
        data := toFloat64Slice(call.Arguments[0].Export())
        fast := int(call.Arguments[1].ToInteger())
        slow := int(call.Arguments[2].ToInteger())
        signal := int(call.Arguments[3].ToInteger())
        macdLine, signalLine, histogram := indicators.MACD(data, fast, slow, signal)
        val, _ := sb.runtime.ToValue(map[string]interface{}{
            "macd": macdLine, "signal": signalLine, "histogram": histogram,
        })
        return val
    })
    
    // 滚动相关性
    utils.Set("computeRollingCorrelation", func(call goja.FunctionCall) goja.Value {
        seriesA := toFloat64Slice(call.Arguments[0].Export())
        seriesB := toFloat64Slice(call.Arguments[1].Export())
        windows := toIntSlice(call.Arguments[2].Export())
        result := stats.RollingCorrelation(seriesA, seriesB, windows)
        val, _ := sb.runtime.ToValue(result)
        return val
    })
    
    // 相关性矩阵
    utils.Set("computeCorrelationMatrix", func(call goja.FunctionCall) goja.Value {
        series := toFloat64SliceSlice(call.Arguments[0].Export())
        labels := toStringSlice(call.Arguments[1].Export())
        result := stats.CorrelationMatrix(series, labels)
        val, _ := sb.runtime.ToValue(result)
        return val
    })
    
    // 时间序列重采样
    utils.Set("resample", func(call goja.FunctionCall) goja.Value {
        data := call.Arguments[0].Export()
        targetTimeframe := call.Arguments[1].String()
        result := timeseries.Resample(data, targetTimeframe)
        val, _ := sb.runtime.ToValue(result)
        return val
    })
    
    // 百分比变化
    utils.Set("pctChange", func(call goja.FunctionCall) goja.Value {
        data := toFloat64Slice(call.Arguments[0].Export())
        result := stats.PctChange(data)
        val, _ := sb.runtime.ToValue(result)
        return val
    })
    
    // 标准化 (0-1)
    utils.Set("normalize", func(call goja.FunctionCall) goja.Value {
        data := toFloat64Slice(call.Arguments[0].Export())
        result := stats.Normalize(data)
        val, _ := sb.runtime.ToValue(result)
        return val
    })
    
    sb.runtime.Set("utils", utils)
}
```

### 执行入口

```go
// ExecuteSearch 执行 search tool
func (mgr *SandboxManager) ExecuteSearch(ctx context.Context, code string) (interface{}, error) {
    sb := mgr.acquireSandbox()
    defer mgr.releaseSandbox(sb)
    
    // search 只需要 spec，不需要 sdk
    return mgr.executeWithTimeout(ctx, sb, code)
}

// ExecuteCode 执行 execute tool
func (mgr *SandboxManager) ExecuteCode(ctx context.Context, code string) (*DashboardState, error) {
    sb := mgr.acquireSandbox()
    defer mgr.releaseSandbox(sb)
    
    mgr.injectSDK(sb, ctx)
    mgr.injectUtils(sb)
    
    result, err := mgr.executeWithTimeout(ctx, sb, code)
    if err != nil {
        return nil, err
    }
    
    state, err := parseDashboardState(result)
    if err != nil {
        return nil, fmt.Errorf("invalid DashboardState: %w", err)
    }
    return state, nil
}

// ExecuteMutate 执行 mutate tool
func (mgr *SandboxManager) ExecuteMutate(ctx context.Context, code string, currentState *DashboardState) (*DashboardState, error) {
    sb := mgr.acquireSandbox()
    defer mgr.releaseSandbox(sb)
    
    mgr.injectSDK(sb, ctx)
    mgr.injectUtils(sb)
    
    // 注入当前 state 作为参数
    stateVal, _ := sb.runtime.ToValue(stateToMap(currentState))
    
    // 包装为接收 state 参数的函数
    wrappedCode := fmt.Sprintf(`(function() { var fn = %s; return fn(state); })()`, code)
    sb.runtime.Set("state", stateVal)
    
    result, err := mgr.executeWithTimeout(ctx, sb, wrappedCode)
    if err != nil {
        return nil, err
    }
    
    newState, err := parseDashboardState(result)
    if err != nil {
        return nil, fmt.Errorf("invalid DashboardState: %w", err)
    }
    return newState, nil
}

// executeWithTimeout 带超时执行
func (mgr *SandboxManager) executeWithTimeout(ctx context.Context, sb *Sandbox, code string) (interface{}, error) {
    ctx, cancel := context.WithTimeout(ctx, mgr.config.ExecutionTimeout)
    defer cancel()
    
    // 启动中断 goroutine
    done := make(chan struct{})
    go func() {
        select {
        case <-ctx.Done():
            sb.runtime.Interrupt("execution timeout")
        case <-done:
        }
    }()
    
    result, err := sb.runtime.RunString(code)
    close(done)
    
    if err != nil {
        return nil, fmt.Errorf("sandbox execution error: %w", err)
    }
    
    return result.Export(), nil
}
```

### 沙箱池管理

```go
func (mgr *SandboxManager) acquireSandbox() *Sandbox {
    select {
    case sb := <-mgr.pool:
        sb.reset()
        return sb
    default:
        // 池空了，创建新的
        return mgr.createSandbox()
    }
}

func (mgr *SandboxManager) releaseSandbox(sb *Sandbox) {
    select {
    case mgr.pool <- sb:
        // 放回池
    default:
        // 池满了，丢弃
    }
}

func (sb *Sandbox) reset() {
    sb.sdkCalls = 0
    sb.logs = nil
}
```

## async/await 处理方案

goja 不支持原生 async/await。两种处理方式：

### 方案 A：同步 SDK 接口（推荐）

SDK Proxy 对 JS 暴露同步接口。Go 侧在调用实际 API 时仍然并行，但对 goja 来说是阻塞调用。

Agent 生成的代码就不用 async/await：
```javascript
(function() {
  var gold = sdk.market.getKline({ symbol: 'XAUUSD', timeframe: '1D' });
  var btc = sdk.market.getKline({ symbol: 'BTCUSD', timeframe: '1D' });
  return { title: '...', panels: [...] };
})()
```

**问题：** 丢失了 Promise.all 并行能力。

**解决：** 提供 `utils.parallel()` 函数：
```go
utils.Set("parallel", func(call goja.FunctionCall) goja.Value {
    // 接收多个函数，Go 侧并行执行
    fns := call.Arguments
    results := make([]interface{}, len(fns))
    var wg sync.WaitGroup
    
    for i, fn := range fns {
        wg.Add(1)
        go func(idx int, f goja.Value) {
            defer wg.Done()
            callable, _ := goja.AssertFunction(f)
            val, _ := callable(goja.Undefined())
            results[idx] = val.Export()
        }(i, fn)
    }
    wg.Wait()
    
    val, _ := sb.runtime.ToValue(results)
    return val
})
```

Agent 代码变为：
```javascript
(function() {
  var results = utils.parallel(
    function() { return sdk.market.getKline({ symbol: 'XAUUSD', timeframe: '1D' }); },
    function() { return sdk.market.getKline({ symbol: 'BTCUSD', timeframe: '1D' }); },
    function() { return sdk.social.searchTweets({ query: 'gold', limit: 50 }); }
  );
  var gold = results[0], btc = results[1], tweets = results[2];
  return { title: '...', panels: [...] };
})()
```

### 方案 B：代码预处理

在执行前将 Agent 生成的 async/await 代码转换为同步版本：
- `await sdk.xxx()` → `sdk.xxx()`
- `Promise.all([...])` → `utils.parallel(...)`

这个转换可以用正则或简单 AST 完成。

### System Prompt 适配

如果选方案 A，system prompt 中指导 Agent 使用同步风格：
```
## 代码编写规范
- SDK 调用是同步的，直接 var result = sdk.xxx(params)
- 多个独立调用用 utils.parallel() 并行执行
- 不要使用 async/await 或 Promise
```

## 安全检查清单

| 风险 | 缓解措施 |
|---|---|
| 无限循环 | 执行超时 + runtime.Interrupt |
| 内存溢出 | goja 内存监控（需定制）/ V8 isolate 原生支持 |
| 访问宿主文件系统 | 不注入 fs / require / import |
| 网络请求 | 不注入 fetch / XMLHttpRequest，只能通过 sdk proxy |
| 环境变量泄露 | 不注入 process / os |
| SDK 滥用 | 调用次数限制 + 限流 |
| 代码注入 | Agent 代码在独立 runtime，不与其他用户共享 |
| Prompt injection via data | SDK 返回的数据不包含可执行代码，只是 JSON |
