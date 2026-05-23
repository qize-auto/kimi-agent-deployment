# 策略联动失效诊断报告

## 一、问题现象确认

**操作**: 在实时盯盘页面输入股票代码 000001（平安银行），加载后切换策略匹配卡片的策略选择器。

**切换前（均值回归策略）**: 买卖信号卡片显示买入信号（RSI超卖21.9，信号价11.28，时间2026-05-11，止损10.94，止盈11.96）

**切换后（突破交易策略）**: 
- ✅ 策略匹配卡片：正确更新为"突破交易" 74%匹配度
- ✅ K线图策略信号：正确更新为突破交易的8个信号
- ❌ **买卖信号卡片：完全未变化！仍显示均值回归策略的信号内容**

---

## 二、JavaScript 代码分析结果

### 2.1 架构文档中描述的联动函数 - 全部不存在

| 函数/状态 | 代码中存在？ | 说明 |
|-----------|------------|------|
| `startStrategyLinkage()` | ❌ 不存在 | 文档描述的联动启动函数 |
| `findStrategyMatchCard()` | ❌ 不存在 | 文档描述的卡片查找函数 |
| `readCurrentStrategy()` | ❌ 不存在 | 文档描述的策略读取函数 |
| `dispatchStrategyEvent()` | ❌ 不存在 | 文档描述的事件分发函数 |
| `onStrategyChangeEvent()` | ❌ 不存在 | 文档描述的事件接收函数 |
| `refreshSignalCardDOM()` | ❌ 不存在 | 文档描述的DOM刷新函数 |
| `window._signalState` | ❌ 不存在 | 文档描述的全局状态对象 |
| `CustomEvent('strategyChanged')` | ❌ 不存在 | 文档描述的自定义事件 |
| `MutationObserver` (策略监听) | ❌ 不存在 | 只有modulepreload的observer |

**结论**: 整个"三合一监听"联动架构在代码中完全没有实现。

### 2.2 实际代码中的数据流

Monitor.tsx 组件的关键状态变量（`function Lde()`）:

```javascript
[S, A] = C.useState("均值回归")   // S=当前策略, A=设置策略函数
[a, o] = C.useState(null)         // a=API监控数据(含a.signals)
[B, G] = C.useState([])           // B=K线信号数据(通过M6生成)
[j, P] = C.useState(null)         // j=反思数据
```

### 2.3 关键 useEffect 依赖分析

| Effect | 依赖项 | 触发行为 | 策略切换时是否执行 |
|--------|--------|----------|-------------------|
| Effect 1 | `[F]` | 加载自定义策略列表 | ❌ 不执行 |
| Effect 2 | `[S]` | 加载自定义策略元素到 `T` | ✅ 执行，但只影响因子选股 |
| Effect 3 | `[n, X]` | **调用 `X()` 获取API数据** | ❌ **不执行！** |
| Effect 4 | `[n, X]` | 30秒定时刷新 | ❌ 不执行 |
| Effect 5 | `[n]` | 倒计时定时器 | ❌ 不执行 |
| **Effect 6** | **`[S, a]`** | **调用 `M6()` 更新K线信号 `B`** | ✅ **执行** |

### 2.4 X() 函数分析

`X()` 是核心数据获取函数，负责：
1. 调用 `$L(n)` 获取API监控数据
2. 处理买卖信号 `a.signals`，设置 `strategy` 字段（基于当前 `S` 的值）
3. 调用 `M6()` 生成K线信号
4. 通过 `P(Q)` 设置反思数据

**但 `X()` 只在 `n`（股票代码）变化时通过 Effect 3 执行，不会在 `S`（策略）变化时执行。**

### 2.5 策略切换时的实际行为

当用户切换策略时：

1. `A(newStrategy)` 被调用，`S` 更新
2. **Effect 6 执行**: `M6(a.kline, a.indicators, S, T, n)` → K线信号 `B` 更新 ✅
3. **Effect 3 不执行**: `X()` 不会被调用 → API数据 `a` 不更新 ❌
4. 买卖信号卡片渲染 `a.signals`（旧数据）→ **不变化** ❌

---

## 三、根因诊断

### 🔴 核心根因：买卖信号数据流断裂

**问题**: 买卖信号卡片的数据源 `a.signals` 来自API原始数据，当策略切换时，API不会被重新调用，因此信号数据不会更新。

**具体机制**:

```
用户切换策略
  ↓
S 状态更新 (A(newStrategy))
  ↓
Effect [S,a] 执行 → M6() 更新 K线信号 B ✅
  ↓
Effect [n,X] 不执行 → X() 不调用 → API 不重新获取 ❌
  ↓
a.signals 保持旧数据 → 买卖信号卡片不更新 ❌
```

### 🟡 次要问题：联动架构完全未实现

架构文档中描述的 MutationObserver + CustomEvent 联动机制在代码中不存在。整个联动是"伪实现"——只存在于文档中，没有实际代码。

---

## 四、修复建议

### 方案1：推荐 - 在策略切换时重新获取数据（最小改动）

在策略切换时调用 `X()` 重新获取数据。修改 `A` 函数（策略设置函数）：

```javascript
// 修改策略设置函数
const handleStrategyChange = C.useCallback((newStrategy) => {
    A(newStrategy);
    // 策略切换后重新获取数据
    if (n) {
        c(!0);  // 设置加载状态
        X();     // 重新调用数据获取函数
    }
}, [n, X]);

// 在 Nde 组件中使用新的 handler
g.jsx(Nde, {
    selected: S,
    onSelect: handleStrategyChange,  // 替代 Y=>A(Y)
    ...
})
```

### 方案2：客户端信号过滤（无需额外API调用）

如果API返回的信号已经包含所有策略的数据，可以在客户端根据 `S` 进行过滤：

```javascript
// 添加过滤后的信号
const filteredSignals = C.useMemo(() => {
    if (!a || !a.signals) return [];
    // 根据当前策略过滤信号
    return a.signals.filter(sig => 
        sig.strategy === S || sig.strategyType === S
    );
}, [a, S]);

// 在渲染时使用 filteredSignals 替代 a.signals
```

**注意**: 这需要确认API返回的信号数据是否包含策略标识字段。

### 方案3：实现完整的联动架构（文档描述的方式）

按架构文档实现 MutationObserver + CustomEvent 机制：

```javascript
function startStrategyLinkage() {
    const strategyCard = findStrategyMatchCard();
    if (!strategyCard) return;
    
    let debounceTimer;
    const observer = new MutationObserver((mutations) => {
        clearTimeout(debounceTimer);
        debounceTimer = setTimeout(() => {
            const strategy = readCurrentStrategy();
            dispatchStrategyEvent(strategy);
        }, 150);
    });
    
    observer.observe(strategyCard, { 
        childList: true, 
        subtree: true,
        characterData: true 
    });
}

document.addEventListener('strategyChanged', onStrategyChangeEvent);
```

**此方案工作量较大，建议作为长期重构目标。**

---

## 五、总结

| 项目 | 状态 |
|------|------|
| 问题确认 | ✅ 策略切换后买卖信号卡片不更新 |
| 根因定位 | ✅ API数据在策略切换时不重新获取 |
| 联动架构 | ❌ 文档描述的三合一监听机制完全未实现 |
| K线信号更新 | ✅ 通过 useEffect([S,a]) 正确更新 |
| 修复方案 | ✅ 提供3种方案，推荐方案1（最小改动） |

**最可能的成功修复**: 方案1 - 在 `A(newStrategy)` 后调用 `X()` 重新获取数据。
