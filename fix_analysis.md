# 完整问题分析

## 问题1（致命）：onStrategyChangeEvent时机问题

`onStrategyChangeEvent`的MutationObserver监听的是**策略卡片**的变化，但信号表格的更新可能滞后。

```
用户切换策略
    ↓
React更新策略卡片（策略名、匹配度）
    ↓
MutationObserver检测到策略卡片变化 → 立即触发onStrategyChangeEvent
    ↓
replaceHoldingsCard(hdCard) 从DOM提取信号
    ↓
❌ 信号表格还没有更新！提取的是旧策略的信号！
```

**修复**：添加延时，等待React更新信号表格。

## 问题2（严重）：filterSignalsByStrategy逻辑问题

信号表格由React按策略过滤后，提取的信号strategy字段为空（表格只有4列）。
`filterSignalsByStrategy`尝试匹配strategy字段（空）或reason字段（通常不包含策略名），
匹配失败后返回全部信号，导致过滤无效。

**修复**：当信号表格已由React过滤时，不做额外过滤。

## 问题3（中）：innerHTML=''后卡片结构丢失

`hdCard.innerHTML = ''`清空了卡片内容，包括"买卖信号"标题。
虽然已通过targetCard参数绕过，但卡片CSS类名也可能丢失。

**修复**：不清空innerHTML，而是直接替换内容。
