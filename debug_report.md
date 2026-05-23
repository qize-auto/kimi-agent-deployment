
# 策略联动失效问题诊断报告

## 一、现象描述

通过在浏览器中实际操作验证：

1. 访问实时盯盘页面，输入688322开始监控
2. 初始策略为"均值回归"，买卖信号卡片显示"历史信号共 10 条"
3. 点击策略按钮切换到"趋势跟踪"：
   - 策略信号列表从18个变为12个 ✓ （React组件正常更新）
   - 买卖信号卡片仍然显示"历史信号共 10 条" ✗ （未联动更新）
4. 切换到"MACD动量"：
   - 策略信号列表从12个变为13个 ✓ （React组件正常更新）
   - 买卖信号卡片仍然显示"历史信号共 10 条" ✗ （未联动更新）
5. 切换回"均值回归"：
   - 策略信号列表从13个变为18个 ✓ （React组件正常更新）
   - 买卖信号卡片仍然显示"历史信号共 10 条" ✗ （仍未联动更新）

**结论**：策略信号列表（React组件）随策略切换正常更新，但买卖信号卡片（JS IIFE构建）始终未联动更新。

---

## 二、代码架构分析

### 2.1 关键组件

- **买卖信号卡片**: 由 IIFE (立即执行函数) 中的 `replaceHoldingsCard()` 构建，替换原有的"持仓管理"卡片
- **策略信号列表**: React 组件，随策略状态变化自动重新渲染
- **策略联动机制**: `startStrategyLinkage()` 通过 MutationObserver + 自定义事件 + 轮询实现

### 2.2 联动流程

```
用户点击策略按钮
  → 策略卡片DOM变化
  → MutationObserver 检测到变化（防抖150ms）
  → readCurrentStrategy() 读取当前策略名
  → dispatchStrategyEvent(strategyName) 发送自定义事件
  → onStrategyChangeEvent(event) 处理事件
    → 比较新旧策略
    → 从 processedCards 中删除买卖信号卡片引用
    → 清空卡片 innerHTML
    → 调用 replaceHoldingsCard() 重建卡片
```

---

## 三、根因分析

### 问题1：strategyNames 数组与实际UI策略名称不匹配（主要问题）

**代码位置**: `readCurrentStrategy()` 函数

**代码中的策略名称数组**:
```javascript
var strategyNames = ['均值回归', '动量', '趋势', '突破', '震荡', '价值', '成长', '反转', '波动率'];
```

**实际UI中的策略按钮**:
- 均值回归 ✓ (精确匹配)
- 趋势跟踪 ✗ (代码中只有"趋势")
- MACD动量 ✗ (代码中只有"动量")
- 布林带通道 ✗ (不在数组中)
- 因子选股 ✗ (不在数组中)
- 突破交易 ✗ (代码中只有"突破")
- RSI均值回归 ✗ (与"均值回归"不同)
- 均线多头 ✗ (不在数组中)
- 量价齐升 ✗ (不在数组中)
- 支撑反弹 ✗ (不在数组中)
- 超跌反弹 ✗ (不在数组中)
- WR超卖反弹 ✗ (不在数组中)
- 缩量低吸 ✗ (不在数组中)
- 布林中轨突破 ✗ (代码中只有"突破")

**匹配率: 1/14 = 7.1%**

### 问题2：按钮匹配使用精确相等 (===)

在 `readCurrentStrategy()` 中，对 button 元素的匹配代码：
```javascript
for (var b = 0; b < buttons.length; b++) {
  var btnText = buttons[b].textContent.trim();
  for (var n = 0; n < strategyNames.length; n++) {
    if (btnText === strategyNames[n]) return btnText;  // 精确匹配！
  }
}
```

使用 `===` 精确匹配，导致：
- "趋势跟踪" !== "趋势" → 不匹配
- "MACD动量" !== "动量" → 不匹配
- "布林带通道" → 无匹配

### 问题3：信号数据中的 strategy 字段为空

`extractSignalsFromTable()` 提取信号时：
```javascript
strategy: cells[4] ? cells[4].textContent.trim() : ''
```

大多数信号表格没有第5列，所以 `strategy` 字段为空字符串。

这导致 `filterSignalsByStrategy()` 回退到检查 `reason` 是否包含策略名称，但 reason 通常也不包含策略名。

### 问题4：连锁反应导致联动完全失效

当 `readCurrentStrategy()` 返回空字符串时：

1. `dispatchStrategyEvent('')` 发送空策略名
2. `onStrategyChangeEvent` 中 `newStrategy = ''`
3. 如果 `_signalState.currentStrategy` 也是空字符串，则 `'' === ''` → 直接 return
4. 买卖信号卡片不会被重置和重建

即使用户切换到"均值回归"（能匹配的策略），如果 `_signalState.currentStrategy` 已经被之前的空策略更新为空字符串，切换回"均值回归"时应该触发重建。但实际测试中买卖信号卡片仍然没有变化，这可能是因为：

- `onStrategyChangeEvent` 执行了，但 `replaceHoldingsCard()` 重建时提取的信号数据没有变化（因为信号数据来自当前策略表格，而表格已经由React更新了）
- 或者 `processedCards` 的状态问题导致 `replaceHoldingsCard()` 直接 return

### 问题5：processedCards 状态管理问题

在 `onStrategyChangeEvent` 中：
```javascript
processedCards.delete(hdCard);  // 从已处理集合中移除
hdCard.innerHTML = '';           // 清空DOM
replaceHoldingsCard();           // 重新构建
```

但 `replaceHoldingsCard()` 开头检查：
```javascript
if (!hdCard || processedCards.has(hdCard)) return;
```

如果 `processedCards.delete(hdCard)` 没有正确执行（比如 hdCard 引用已失效），则 `replaceHoldingsCard()` 会直接返回。

---

## 四、修复建议

### 修复1：更新 strategyNames 数组（必须）

```javascript
var strategyNames = [
  '均值回归', '趋势跟踪', 'MACD动量', '布林带通道', 
  '因子选股', '突破交易', 'RSI均值回归', '均线多头',
  '量价齐升', '支撑反弹', '超跌反弹', 'WR超卖反弹',
  '缩量低吸', '布林中轨突破'
];
```

### 修复2：改用包含匹配而非精确匹配

```javascript
// 原代码（精确匹配）
if (btnText === strategyNames[n]) return btnText;

// 修复后（双向包含匹配）
if (btnText === strategyNames[n] || 
    btnText.indexOf(strategyNames[n]) >= 0 || 
    strategyNames[n].indexOf(btnText) >= 0) return btnText;
```

### 修复3：确保信号数据携带策略标识

在信号提取时，从当前策略标题区域读取策略名称并附加到每个信号：

```javascript
// 在 replaceHoldingsCard() 中添加
var strategyTitle = document.querySelector('.strategy-signals-title');
var currentStrategyName = strategyTitle ? strategyTitle.textContent.trim() : '';

sigs.forEach(function(s) {
  if (!s.strategy) s.strategy = currentStrategyName;
});
```

### 修复4：防御性处理空策略名

```javascript
function onStrategyChangeEvent(event) {
  var newStrategy = event.detail && event.detail.strategy ? event.detail.strategy : '';
  if (!newStrategy) return;  // 忽略空策略名
  if (newStrategy === window._signalState.currentStrategy) return;
  // ...
}
```

---

## 五、验证测试结果总结

| 策略 | 策略信号列表 | 买卖信号卡片 | readCurrentStrategy()预期返回值 |
|------|------------|------------|------------------------------|
| 均值回归 | 18个信号 ✓ | 10条（未变）✗ | "均值回归" |
| 趋势跟踪 | 12个信号 ✓ | 10条（未变）✗ | ""（无法匹配） |
| MACD动量 | 13个信号 ✓ | 10条（未变）✗ | ""（无法匹配） |
| 均值回归（切回） | 18个信号 ✓ | 10条（未变）✗ | "均值回归" |

即使切换回能匹配的"均值回归"，买卖信号卡片仍未更新，说明联动机制存在多重问题。
