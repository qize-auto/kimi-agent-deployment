# 买卖信号联动失效 - 深度诊断与修复方案

## 一、用户诉求拆解

| # | 诉求 | 现状 | 期望 |
|---|------|------|------|
| 1 | 切换策略时买卖信号联动 | 切换策略后信号卡片内容完全不变 | 切换策略后显示对应策略的最新信号 |
| 2 | 时间显示 | 显示历史日期(2026-05-07) | 实时盯盘显示"盘中实时" |
| 3 | 不要越改越乱 | 已有多处修改，代码复杂 | 保持现有功能稳定 |

## 二、浏览器实测发现

### 2.1 策略选择器是 `<button>` 不是 `<select>`！

```
[13]<button 均值回归/>  ← 按钮！不是下拉框
```

**致命影响**: `readCurrentStrategy()` 方法1查找 `<select>` 元素 → 找不到 → 降级到方法2 → 可能也找不到 → 返回空字符串 → 过滤无效。

### 2.2 买卖信号显示"当前无活跃买卖信号"

最新信号: 卖出 86.50 (2026-05-07)
当前价: 118.15
偏差: (118.15 - 86.50) / 86.50 = 36.6% > 15%
→ 过期检测触发 → 显示"无活跃信号"

**问题**: 过期检测阈值15%过于严格，正常波动就会被标记过期。

### 2.3 信号表格标题含策略名

```
策略信号 (18个) — 均值回归
```

React已经按策略过滤了信号表格！我们的 `allSignalsCache` 缓存的是当前策略的信号，切换策略后信号表格会更新，但缓存不会自动刷新。

### 2.4 WeakSet不可删除

```javascript
processedCards = new WeakSet();  // 无delete方法！
```

策略切换后无法清除标记 → `replaceHoldingsCard()` 被阻挡 → 卡片不会重建。

## 三、根因总结

```
用户切换策略
    ↓
React更新信号表格（显示新策略的信号）
    ↓
我们的联动代码:
    1. readCurrentStrategy() 返回空（找不到button的值）
    2. filterSignalsByStrategy(sigs, '') 返回全部（无过滤）
    3. processedCards WeakSet 阻止 replaceHoldingsCard() 执行
    4. allSignalsCache 缓存旧策略的信号
    ↓
买卖信号卡片内容 → 完全不变 ❌
```

## 四、修复方案

### 修改1：WeakSet → 普通Object（核心修复）

```javascript
// 修改前
var processedCards = new WeakSet();

// 修改后
var processedCards = { _cards: [] };
processedCards.has = function(card) {
    return this._cards.indexOf(card) >= 0;
};
processedCards.add = function(card) {
    if (this._cards.indexOf(card) < 0) this._cards.push(card);
};
processedCards.delete = function(card) {
    var idx = this._cards.indexOf(card);
    if (idx >= 0) this._cards.splice(idx, 1);
};
```

### 修改2：联动逻辑重写

策略切换时：
1. MutationObserver检测到策略卡片变化
2. 读取新策略名（支持button元素）
3. 删除processedCards中买卖信号卡片的标记
4. 清空买卖信号卡片innerHTML
5. 重新执行 replaceHoldingsCard()
6. 重新从DOM提取信号（获取新策略的信号）

### 修改3：readCurrentStrategy支持button

```javascript
function readCurrentStrategy() {
    // ...现有方法1(select)和方法2(heading)...
    
    // 方法3: 查找button元素
    var buttons = strategyCard.querySelectorAll('button');
    for (var i = 0; i < buttons.length; i++) {
        var text = buttons[i].textContent.trim();
        if (text && strategyNames.indexOf(text) >= 0) return text;
    }
    
    // 方法4: 从卡片heading文本匹配
    var heading = strategyCard.querySelector('h3, h4, .font-heading');
    if (heading) {
        var htext = heading.textContent.trim();
        for (var j = 0; j < strategyNames.length; j++) {
            if (htext.indexOf(strategyNames[j]) >= 0) return strategyNames[j];
        }
    }
    return '';
}
```

### 修改4：过期检测阈值调整

```javascript
// 修改前: 偏离15%就过期
if (priceDeviation > 0.15) { isSignalStale = true; }

// 修改后: 偏离50%或突破止损才过期
if (priceDeviation > 0.50) { isSignalStale = true; }
```

### 修改5：时间显示（已修复）

```javascript
var timeDisplay = '盘中实时';  // 已确认修复
```

## 五、修改清单

| # | 修改 | 位置 | 说明 |
|---|------|------|------|
| 1 | processedCards WeakSet→Object | State区域 | 添加delete方法 |
| 2 | readCurrentStrategy() 增强 | 联动函数区 | 支持button、heading匹配 |
| 3 | onRouteChange() 重置processedCards | 路由切换 | 切换页面时清空标记 |
| 4 | 过期检测阈值 15%→50% | replaceHoldingsCard | 减少误判 |
| 5 | 时间显示修复 | replaceHoldingsCard | 已确认 |

## 六、不影响的部分

- ✅ removeStarCards() 系列
- ✅ setupReflectionFold()
- ✅ enhanceRealtimeCard()
- ✅ syncCardHeight()
- ✅ assets/index-97gBSAr5.js
