
# 策略联动完整数据流分析报告

## 一、完整数据流图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            阶段1: 页面初始化 / 信号提取                               │
└─────────────────────────────────────────────────────────────────────────────────────┘

 [页面加载 / hashchange / MutationObserver]
         │
         ▼
 ┌───────────────┐     ┌──────────────────────┐     ┌─────────────────────────────┐
 │   runAll()    │────▶│  replaceHoldingsCard() │────▶│  信号提取 (两条路径, 二选一)  │
 └───────────────┘     └──────────────────────┘     └─────────────────────────────┘
                                                             │
                                    ┌────────────────────────┴────────────────────────┐
                                    ▼                                                 ▼
                    ┌─────────────────────────┐                    ┌──────────────────────────┐
                    │  Path A: Flex Items提取  │                    │  Path B: Table Rows提取   │
                    │  .flex.items-center.gap-2 │                    │  extractSignalsFromTable()│
                    │                         │                    │                          │
                    │  spans[0]=type (买入/卖出)│                    │  cells[0]=type (买入/卖出)│
                    │  spans[1]=time           │                    │  cells[1]=time            │
                    │  spans[2]=reason         │                    │  cells[2]=reason          │
                    │  spans[3]=price          │                    │  cells[3]=price           │
                    │  spans[4]=strategy  ⚠️   │                    │  cells[4]=strategy   ⚠️   │
                    │  (需要>=5个span)         │                    │  (需要>=5个td)            │
                    └─────────────────────────┘                    └──────────────────────────┘
                                    │                                                 │
                                    └────────────────────────┬────────────────────────┘
                                                             ▼
                                              ┌───────────────────────────┐
                                              │   sigs[] (信号数组)        │
                                              │   每个元素: {type, time,  │
                                              │   reason, price, strategy}│
                                              └───────────────────────────┘
                                                             │
                             ┌───────────────────────────────┼───────────────────────────────┐
                             ▼                               ▼                               ▼
                    ┌─────────────────┐           ┌────────────────────┐           ┌─────────────────────┐
                    │ readCurrentStrategy()│      │ filterSignalsByStrategy()│      │  构建卡片DOM         │
                    │   (读取当前策略名)  │      │   (策略过滤)         │      │   (innerHTML=''重建) │
                    └─────────────────┘           └────────────────────┘           └─────────────────────┘
                             │                               ▲                               │
                             ▼                               │                               ▼
                    ┌─────────────────┐           ┌────────────────────┐           ┌─────────────────────┐
                    │ findStrategyMatchCard()      │ 匹配逻辑:          │           │ processedCards.add()│
                    │   1. findTextNode('策略匹配') │ 1. strategy===精确匹配│          │                     │
                    │   2. 正则 \d+%\s*匹配度   │ 2. reason.includes()│          │  ⚠️ 卡片标记为已处理  │
                    └─────────────────┘           │ 3. 无匹配则返回全部  │           └─────────────────────┘
                             │                    └────────────────────┘
                             ▼                               │
                    ┌─────────────────┐                       │
                    │ 策略卡片内搜索:   │◄──────────────────────┘
                    │ 1. <select> value│
                    │ 2. <button> text │
                    │ 3. <h3/h4> text  │
                    │ 4. 任意元素精确匹配│
                    └─────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ currentStrategy  │
                    │ (策略名字符串)   │
                    └─────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            阶段2: 策略切换监听机制                                   │
└─────────────────────────────────────────────────────────────────────────────────────┘

 [init() 启动]
     │
     ▼
 ┌─────────────────────┐
 │ startStrategyLinkage()│
 └─────────────────────┘
     │
     ├──► [监听1] document.addEventListener('strategyChanged', onStrategyChangeEvent)
     │
     ├──► [监听2] MutationObserver 附加到策略卡片 ⚠️
     │           (findStrategyMatchCard() 可能返回null)
     │
     └──► [监听3] setInterval(2000ms) 轮询 ⚠️
                 (仅location.hash.includes('monitor')时生效)


┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            阶段3: 策略切换响应流程                                   │
└─────────────────────────────────────────────────────────────────────────────────────┘

 [用户点击策略按钮]
     │
     ▼
 [React 更新DOM]
     │
     ├──► [路径A] MutationObserver触发
     │           └──► readCurrentStrategy() ──► dispatchStrategyEvent(name)
     │
     └──► [路径B] 2秒后setInterval轮询捕获
                 └──► readCurrentStrategy() ──► dispatchStrategyEvent(name)
                                    │
                                    ▼
                    ┌─────────────────────────────────────┐
                    │  CustomEvent('strategyChanged')      │
                    │  detail: { strategy: newStrategyName }│
                    └─────────────────────────────────────┘
                                    │
                                    ▼
                    ┌─────────────────────────────────────┐
                    │   onStrategyChangeEvent(event)       │
                    │                                      │
                    │  1. newStrategy = event.detail.strategy│
                    │  2. if (newStrategy === current) return│
                    │  3. currentStrategy = newStrategy     │
                    │  4. 查找买卖信号卡片 (.data-card)      │
                    │     条件是 textContent.includes('买卖信号')│
                    │  5. processedCards.delete(hdCard)     │
                    │  6. hdCard.innerHTML = ''    ⚠️⚠️⚠️  │
                    │  7. replaceHoldingsCard()    ⚠️⚠️⚠️  │
                    └─────────────────────────────────────┘
                                    │
                                    ▼
                    ┌─────────────────────────────────────┐
                    │   replaceHoldingsCard() 内部执行:    │
                    │                                      │
                    │  hdNode = findTextNode('持仓管理')     │
                    │  if (!hdNode)                        │
                    │    hdNode = findTextNode('买卖信号')   │
                    │  if (!hdNode) return;    ⚠️⚠️⚠️      │
                    │  // ←─ 这里返回了! 卡片不会重建       │
                    │  // 因为innerHTML=''后文本不存在了     │
                    └─────────────────────────────────────┘
```

---

## 二、每个转换节点的详细分析

### 节点1: extractSignalsFromTable() (第112-141行)

**输入**: DOM中的 `<table>` 元素
**输出**: `sigs[]` 数组, 每个元素 `{type, time, reason, price, strategy}`

**转换逻辑**:
```javascript
document.querySelectorAll('table tbody tr, table tr').forEach(function(tr) {
  var cells = tr.querySelectorAll('td');
  if (cells.length >= 4) {              // ← 检查至少4列
    var type = cells[0].textContent.trim();
    if (type === '买入' || type === '卖出') {
      sigs.push({
        type: type,
        time: cells[1] && cells[1].textContent.trim(),
        reason: cells[2] && cells[2].textContent.trim(),
        price: cells[3] && cells[3].textContent.trim(),
        strategy: cells[4] ? cells[4].textContent.trim() : ''  // ← 第5列
      });
    }
  }
});
```

**关键发现**: 
- 条件 `cells.length >= 4` 只保证有4列，但 `strategy` 取自 `cells[4]`（第5列）
- 如果表格实际只有4列：`type, time, reason, price` → `cells[4]` = `undefined` → `strategy = ''`
- 去重检查仅基于 `type + time` 组合

**风险等级**: 🔴 **高风险**

---

### 节点2: Flex Items信号提取 (第572-586行)

**输入**: DOM中的 `.flex.items-center.gap-2` 元素
**输出**: `sigs[]` 数组（与table提取相同结构）

**转换逻辑**:
```javascript
document.querySelectorAll('.flex.items-center.gap-2').forEach(function(it) {
  var spans = it.querySelectorAll(':scope > span');
  if (spans.length >= 4) {              // ← 检查至少4个span
    var type = spans[0].textContent.trim();
    if (type === '买入' || type === '卖出') {
      sigs.push({
        type: type,
        time: spans[1].textContent.trim(),
        reason: spans[2].textContent.trim(),
        price: spans[3].textContent.trim(),
        strategy: spans[4] ? spans[4].textContent.trim() : ''  // ← 第5个span
      });
    }
  }
});
```

**关键发现**:
- 与table提取相同的列数问题：`spans.length >= 4` 但取 `spans[4]`
- 优先使用flex提取（第589行 `if (sigs.length === 0)` 才fallback到table）
- `:scope > span` 只取直接子元素

**风险等级**: 🔴 **高风险**（与节点1相同问题）

---

### 节点3: findStrategyMatchCard() (第146-161行)

**输入**: `document.body`（全文搜索）
**输出**: 策略卡片的DOM元素 或 `null`

**转换逻辑**:
```javascript
function findStrategyMatchCard() {
  // Method 1: 精确匹配"策略匹配"文本
  var node = findTextNode('策略匹配');
  if (node) {
    var card = findParentCard(node);
    if (card) return card;
  }
  // Method 2: Fallback - 匹配 "xx% 匹配度" 格式
  var walker = document.createTreeWalker(document.body, 4, null, false);
  var n;
  while (n = walker.nextNode()) {
    if (/\d+%\s*匹配度/.test(n.textContent)) {
      return findParentCard(n);
    }
  }
  return null;
}
```

**关键发现**:
- `findTextNode()` 使用 `document.createTreeWalker(document.body, 4, ...)` 全量遍历所有文本节点
- 每次调用都遍历整个DOM树，性能开销大（O(n)，n=文本节点数）
- 如果页面没有"策略匹配"文本且没有"xx% 匹配度"格式，返回null
- `findParentCard()` 向上遍历到包含 `className.indexOf('data-card') >= 0` 的元素

**风险等级**: 🟡 **中风险**（功能正确但性能差）

---

### 节点4: readCurrentStrategy() (第164-206行)

**输入**: 策略卡片DOM元素
**输出**: 策略名字符串（如"均值回归"）或 `''`

**转换逻辑**:
```javascript
function readCurrentStrategy() {
  var strategyCard = findStrategyMatchCard();
  if (!strategyCard) return '';

  var strategyNames = ['均值回归', '动量', '趋势', '突破', '震荡', '价值', '成长', '反转', '波动率'];
  // 硬编码策略名列表 ←─── ⚠️

  // Method 1: <select> value
  var select = strategyCard.querySelector('select');
  if (select) { ... }

  // Method 2: <button> textContent (精确匹配)
  var buttons = strategyCard.querySelectorAll('button');
  for (...) {
    if (btnText === strategyNames[n]) return btnText;  // ← 精确匹配
  }

  // Method 3: heading elements
  var headingEls = strategyCard.querySelectorAll('h3, h4, .font-heading');

  // Method 4: 任意元素精确匹配
  var allEls = strategyCard.querySelectorAll('*');
  for (...) {
    if (t === strategyNames[p]) return t;  // ← 精确匹配
  }
  return '';
}
```

**关键发现**:
- **策略名列表硬编码**：第167行 `['均值回归', '动量', '趋势', '突破', '震荡', '价值', '成长', '反转', '波动率']`
- 如果实际策略名是"趋势跟踪"、"双均线"等不在列表中的名字 → 返回 `''`
- Method 2和Method 4都是**精确匹配**（`===`），不是包含匹配
- 如果按钮文本是"  均值回归  "（带空格），trim后仍可匹配
- 但如果按钮文本是"均值回归策略"或"选择：均值回归"，精确匹配失败

**风险等级**: 🟡 **中风险**（硬编码限制扩展性）

---

### 节点5: filterSignalsByStrategy() (第221-232行)

**输入**: `sigs[]`（全部信号），`strategy`（当前策略名）
**输出**: 过滤后的信号数组

**转换逻辑**:
```javascript
function filterSignalsByStrategy(sigs, strategy) {
  if (!strategy || !sigs || sigs.length === 0) return sigs;  // ← 短路返回原数组
  var filtered = [];
  for (var i = 0; i < sigs.length; i++) {
    // 条件1: strategy字段精确匹配
    if (sigs[i].strategy && sigs[i].strategy === strategy) {
      filtered.push(sigs[i]);
    } 
    // 条件2: reason字段包含策略名（子串匹配）
    else if (sigs[i].reason && sigs[i].reason.indexOf(strategy) >= 0) {
      filtered.push(sigs[i]);
    }
  }
  return filtered.length > 0 ? filtered : sigs;  // ← fallback到全部
}
```

**调用处**（第614-619行）:
```javascript
var currentStrategy = readCurrentStrategy();
window._signalState.currentStrategy = currentStrategy;
if (currentStrategy && sigs.length > 0) {
  var filtered = filterSignalsByStrategy(sigs, currentStrategy);
  if (filtered.length > 0 && filtered.length < sigs.length) {  // ← 关键条件
    sigs = filtered;
  }
}
```

**关键发现**:
- 如果所有信号的 `strategy` 字段为空（表格只有4列的情况）:
  - `sigs[i].strategy` 为空falsy值 → 条件1不成立
  - fallback到条件2：`reason.indexOf(strategy)` 
  - 如果reason也不包含策略名 → 不加入filtered
  - 最终 `filtered.length === 0`
  - `filtered.length > 0` 为false → 保留全部sigs（不过滤）
  - **结果：策略切换不生效，始终显示全部信号**

- 即使信号有strategy字段，但如果当前策略的信号数量和全部信号数量相同:
  - `filtered.length < sigs.length` 为false → 保留全部sigs
  - 这是一个设计意图：如果过滤后数量没减少，显示全部

**风险等级**: 🟡 **中风险**（与节点1/2的strategy字段缺失组合成严重问题）

---

### 节点6: onStrategyChangeEvent() (第311-336行)

**输入**: `CustomEvent` 对象（含 `detail.strategy`）
**输出**: 副作用（DOM更新）

**转换逻辑**:
```javascript
function onStrategyChangeEvent(event) {
  var newStrategy = '';
  if (event.detail && event.detail.strategy) {
    newStrategy = event.detail.strategy;
  }
  if (newStrategy === window._signalState.currentStrategy) return;  // ← 相同则忽略
  window._signalState.currentStrategy = newStrategy;

  // 查找买卖信号卡片
  var hdCard = null;
  var cards = document.querySelectorAll('.data-card');
  for (var c = 0; c < cards.length; c++) {
    if (cards[c].textContent.indexOf('买卖信号') >= 0) {
      hdCard = cards[c];
      break;
    }
  }
  if (hdCard) {
    processedCards.delete(hdCard);   // ← 从已处理集合移除
    hdCard.innerHTML = '';            // ← ⚠️ 清空内容
    replaceHoldingsCard();            // ← ⚠️ 重建卡片
  }
}
```

**关键发现**:
- `processedCards.delete(hdCard)` 正确移除了卡片的"已处理"标记
- `hdCard.innerHTML = ''` 清空了所有内容，包括之前设置的 `'买卖信号'` 标题文本
- 然后调用 `replaceHoldingsCard()`

**风险等级**: 🔴 **严重断裂点**（见下方分析）

---

### 节点7: replaceHoldingsCard() 在重建时的执行 (第552-848行)

**输入**: 全局DOM状态
**输出**: 重建的买卖信号卡片DOM

**关键开头代码**（第554-559行）:
```javascript
function replaceHoldingsCard() {
  var hdNode = findTextNode('持仓管理');
  if (!hdNode) hdNode = findTextNode('买卖信号');
  if (!hdNode) return;   // ← ⚠️⚠️⚠️ 这里会直接return!

  var hdCard = findParentCard(hdNode);
  if (!hdCard || processedCards.has(hdCard)) return;
  // ...
}
```

**断裂分析**:
1. 第一次调用 `replaceHoldingsCard()` 时，页面有"持仓管理"文本 → 找到卡片 → 重建 → 设置标题为"买卖信号"
2. `processedCards.add(hdCard)` 标记为已处理
3. 用户切换策略 → `onStrategyChangeEvent` 触发
4. `hdCard.innerHTML = ''` → 卡片变为空，"买卖信号"文本消失
5. 调用 `replaceHoldingsCard()`
6. `findTextNode('持仓管理')` → 找不到（已被替换）
7. `findTextNode('买卖信号')` → 找不到（innerHTML已被清空）
8. `if (!hdNode) return;` → **函数直接返回，卡片不重建！**

**风险等级**: 🔴🔴🔴 **致命断裂点 - 卡片永远无法在策略切换后重建**

---

### 节点8: dispatchStrategyEvent() (第209-218行)

**输入**: 策略名字符串
**输出**: 副作用（派发CustomEvent）

**转换逻辑**:
```javascript
function dispatchStrategyEvent(strategyName) {
  var event;
  if (document.createEvent) {
    event = document.createEvent('CustomEvent');
    event.initCustomEvent('strategyChanged', true, true, { strategy: strategyName });
  } else {
    event = { type: 'strategyChanged', detail: { strategy: strategyName } };
  }
  document.dispatchEvent(event);
}
```

**关键发现**:
- 使用 `document.createEvent` + `initCustomEvent` 是旧版API
- 现代浏览器应使用 `new CustomEvent('strategyChanged', { detail: { strategy: strategyName } })`
- `initCustomEvent` 的第四个参数是 `detail`，但旧版IE可能不支持
- `event` 对象在 else 分支中是一个普通对象，不是 Event 实例，`document.dispatchEvent(event)` 可能会抛错
- 但由于 `document.createEvent` 在所有现代浏览器都存在，else 分支实际上不会执行

**风险等级**: 🟢 **低风险**（兼容性写法，功能正常）

---

### 节点9: startStrategyLinkage() 中的 MutationObserver 附件 (第339-371行)

**输入**: 无
**输出**: 副作用（附加MutationObserver）

**关键代码**:
```javascript
function startStrategyLinkage() {
  var state = window._signalState;
  if (state.strategyObserving) return;  // ← 防止重复附加
  state.strategyObserving = true;

  document.addEventListener('strategyChanged', onStrategyChangeEvent);

  var strategyCard = findStrategyMatchCard();
  if (strategyCard) {  // ← 可能为null
    var observer = new MutationObserver(function() {
      clearTimeout(strategyDebounceTimer);
      strategyDebounceTimer = setTimeout(function() {
        var newStrategy = readCurrentStrategy();
        if (newStrategy && newStrategy !== state.currentStrategy) {
          dispatchStrategyEvent(newStrategy);
        }
      }, 150);
    });
    observer.observe(strategyCard, { childList: true, subtree: true, characterData: true });
  }

  // 轮询fallback（每2秒）
  setInterval(function() {
    if (!location.hash.includes('monitor')) return;
    var newStrategy = readCurrentStrategy();
    if (newStrategy && newStrategy !== state.currentStrategy) {
      dispatchStrategyEvent(newStrategy);
    }
  }, 2000);
}
```

**关键发现**:
- `startStrategyLinkage()` 在 `init()` 中被调用，此时React可能尚未渲染策略卡片
- `findStrategyMatchCard()` 可能返回 null → MutationObserver 不附加
- 但有 setInterval(2000ms) 轮询作为 fallback
- `strategyObserving` 标记一旦设为 true 永不重置，即使策略卡片后来出现也不会重新附加Observer
- 轮询只在 `location.hash.includes('monitor')` 时生效

**风险等级**: 🟡 **中风险**（MutationObserver可能无法附加，但有轮询fallback）

---

## 三、数据断裂点汇总

| # | 断裂点位置 | 严重等级 | 描述 | 触发条件 |
|---|-----------|---------|------|---------|
| 1 | `onStrategyChangeEvent` → `replaceHoldingsCard` | 🔴 **致命** | innerHTML清空后，findTextNode找不到'买卖信号'，卡片永不重建 | 每次策略切换时必触发 |
| 2 | `extractSignalsFromTable` / Flex提取 | 🔴 **严重** | 检查`>=4`列但取第5列(cells[4])，表格只有4列时strategy全为空 | 信号表格只有4列时 |
| 3 | `filterSignalsByStrategy` 短路逻辑 | 🟡 **中** | strategy为空且reason不含策略名时返回全部sigs，过滤失效 | 与断裂点2同时触发 |
| 4 | `readCurrentStrategy` 硬编码策略名 | 🟡 **中** | 策略名列表写死在代码中，新策略名无法识别 | 使用不在列表中的策略名 |
| 5 | `startStrategyLinkage` Observer附件 | 🟡 **中** | init()时策略卡片可能未渲染，Observer无法附加 | 页面加载时策略卡片尚未渲染 |
| 6 | `findTextNode` 全量DOM遍历 | 🟡 **中** | 每次调用都遍历整个body的文本节点，性能差 | 每次信号提取/策略读取 |
| 7 | `processedCards` 数组实现 | 🟢 **低** | 用数组模拟Set，卡片被移除后引用仍在数组中 | DOM元素被删除时 |

---

## 四、修复建议

### 修复1: 致命断裂点 - onStrategyChangeEvent中的卡片重建（优先级：P0）

**问题**: `hdCard.innerHTML = ''` 后 `replaceHoldingsCard()` 的 `findTextNode('买卖信号')` 找不到文本

**方案A**（推荐）: 让 `replaceHoldingsCard` 接受可选的card参数:
```javascript
function replaceHoldingsCard(targetCard) {
  var hdNode, hdCard;
  if (targetCard) {
    hdCard = targetCard;
  } else {
    hdNode = findTextNode('持仓管理');
    if (!hdNode) hdNode = findTextNode('买卖信号');
    if (!hdNode) return;
    hdCard = findParentCard(hdNode);
  }
  if (!hdCard || processedCards.has(hdCard)) return;
  // ... rest of the function
}

// onStrategyChangeEvent中:
if (hdCard) {
  processedCards.delete(hdCard);
  hdCard.innerHTML = '';
  replaceHoldingsCard(hdCard);  // 传入card引用
}
```

**方案B**: 在清空innerHTML前，先添加一个标记文本:
```javascript
if (hdCard) {
  processedCards.delete(hdCard);
  hdCard.innerHTML = '<span style="display:none">买卖信号</span>';  // 保留标记
  replaceHoldingsCard();
}
```

---

### 修复2: 信号提取的列数检查（优先级：P1）

**问题**: 检查 `>= 4` 但取第5列

**修复**:
```javascript
// Table提取 - 将 >= 4 改为 >= 5
if (cells.length >= 5) {  // 确保有strategy列

// Flex提取 - 将 >= 4 改为 >= 5
if (spans.length >= 5) {  // 确保有strategy span
```

或者更安全的做法：不严格要求第5列，而是从reason中解析策略名:
```javascript
strategy: cells[4] ? cells[4].textContent.trim() : 
          (cells[2] ? extractStrategyFromReason(cells[2].textContent.trim()) : '')
```

---

### 修复3: readCurrentStrategy 策略名列表（优先级：P1）

**问题**: 硬编码策略名，不支持扩展

**修复** - 使用包含匹配替代精确匹配:
```javascript
function readCurrentStrategy() {
  var strategyCard = findStrategyMatchCard();
  if (!strategyCard) return '';

  // 从策略卡片中提取所有文本
  var cardText = strategyCard.textContent;

  // 扩展的策略名列表（支持动态添加）
  var strategyNames = window._signalState.availableStrategies || 
    ['均值回归', '动量', '趋势', '突破', '震荡', '价值', '成长', '反转', '波动率', '趋势跟踪', '双均线'];

  // 使用包含匹配（支持"均值回归策略"这样的格式）
  for (var i = 0; i < strategyNames.length; i++) {
    if (cardText.indexOf(strategyNames[i]) >= 0) return strategyNames[i];
  }

  // Fallback: 从select/button的value/text中读取
  var select = strategyCard.querySelector('select');
  if (select && select.value) return select.value.trim();

  var activeBtn = strategyCard.querySelector('button[data-active="true"], button.active, button[aria-pressed="true"]');
  if (activeBtn) return activeBtn.textContent.trim();

  return '';
}
```

---

### 修复4: startStrategyLinkage Observer延迟附加（优先级：P2）

**问题**: init()时策略卡片可能未渲染

**修复**:
```javascript
function startStrategyLinkage() {
  var state = window._signalState;
  if (state.strategyObserving) return;
  state.strategyObserving = true;

  document.addEventListener('strategyChanged', onStrategyChangeEvent);

  // 延迟附加MutationObserver，等待React渲染完成
  function tryAttachObserver() {
    var strategyCard = findStrategyMatchCard();
    if (strategyCard) {
      if (state._strategyObserver) {
        state._strategyObserver.disconnect();
      }
      state._strategyObserver = new MutationObserver(function() {
        clearTimeout(strategyDebounceTimer);
        strategyDebounceTimer = setTimeout(function() {
          var newStrategy = readCurrentStrategy();
          if (newStrategy && newStrategy !== state.currentStrategy) {
            dispatchStrategyEvent(newStrategy);
          }
        }, 150);
      });
      state._strategyObserver.observe(strategyCard, { childList: true, subtree: true, characterData: true });
      return true;
    }
    return false;
  }

  // 尝试多次附加
  if (!tryAttachObserver()) {
    var attachAttempts = 0;
    var attachInterval = setInterval(function() {
      attachAttempts++;
      if (tryAttachObserver() || attachAttempts > 20) {  // 最多尝试20次(10秒)
        clearInterval(attachInterval);
      }
    }, 500);
  }

  // 保留轮询作为最后fallback
  setInterval(function() { ... }, 2000);
}
```

---

### 修复5: 使用缓存优化 findTextNode（优先级：P2）

**问题**: 每次调用都全量遍历DOM

**修复** - 添加缓存机制:
```javascript
// 缓存常用文本节点的查找结果
var textNodeCache = {};
function findTextNode(text) {
  // 检查缓存（如果节点仍在DOM中且文本匹配）
  var cached = textNodeCache[text];
  if (cached && cached.parentElement && cached.textContent.trim() === text) {
    return cached;
  }

  var w = document.createTreeWalker(document.body, 4, null, false), n;
  while (n = w.nextNode()) {
    if (n.textContent.trim() === text) {
      textNodeCache[text] = n;  // 缓存结果
      return n;
    }
  }
  textNodeCache[text] = null;
  return null;
}

// 在DOM变化时清除缓存
function clearTextNodeCache() {
  textNodeCache = {};
}
```

---

### 修复6: 增强信号缓存与刷新（优先级：P1）

**问题**: 策略切换后信号没有从state中过滤

**修复** - 在 `onStrategyChangeEvent` 中利用已缓存的全量信号:
```javascript
function onStrategyChangeEvent(event) {
  var newStrategy = event.detail && event.detail.strategy ? event.detail.strategy : '';
  if (newStrategy === window._signalState.currentStrategy) return;
  window._signalState.currentStrategy = newStrategy;

  var hdCard = window._signalState.hdCardRef;
  if (!hdCard || !document.body.contains(hdCard)) {
    // 如果引用失效，重新查找
    var cards = document.querySelectorAll('.data-card');
    for (var c = 0; c < cards.length; c++) {
      if (cards[c].querySelector('.signal-card-equal-height') || 
          cards[c].classList.contains('signal-card-equal-height')) {
        hdCard = cards[c];
        window._signalState.hdCardRef = hdCard;
        break;
      }
    }
  }

  if (hdCard) {
    processedCards.delete(hdCard);
    hdCard.innerHTML = '';
    // 如果已有全量信号缓存，直接使用
    if (window._signalState.allSignals.length > 0) {
      // 不重新提取，直接重建（信号已在replaceHoldingsCard中过滤）
    }
    replaceHoldingsCard(hdCard);
  }
}
```

---

## 五、修复优先级排序

| 优先级 | 修复项 | 影响 | 工作量 |
|-------|-------|------|-------|
| **P0** | 修复1: onStrategyChangeEvent卡片重建 | 致命 - 策略切换完全无效 | 小（传参即可） |
| **P1** | 修复2: 信号提取列数检查 | 严重 - 策略过滤永不生效 | 小（改数字） |
| **P1** | 修复3: readCurrentStrategy硬编码 | 严重 - 新策略不支持 | 中（改匹配逻辑） |
| **P1** | 修复6: 信号缓存优化 | 中 - 提升响应速度 | 小 |
| **P2** | 修复4: Observer延迟附加 | 低 - 有轮询fallback | 中 |
| **P2** | 修复5: findTextNode缓存 | 低 - 性能优化 | 小 |

---

## 六、数据流完整性验证

修复后的预期数据流:

```
用户切换策略
  → React更新策略卡片DOM
    → MutationObserver触发 (或轮询捕获)
      → readCurrentStrategy() → 读取新策略名 ✅
      → dispatchStrategyEvent(name) → 派发事件 ✅
        → onStrategyChangeEvent(event) 
          → 获取 event.detail.strategy ✅
          → 与currentStrategy比较，不同则继续 ✅
          → window._signalState.currentStrategy = newStrategy ✅
          → 通过hdCardRef或.class查找卡片 ✅ (修复后)
          → processedCards.delete(hdCard) ✅
          → hdCard.innerHTML = '' ✅
          → replaceHoldingsCard(hdCard) → 传入card引用 ✅ (修复后)
            → 不依赖findTextNode，直接使用传入的card ✅
            → extractSignalsFromTable() → 提取含strategy字段的信号 ✅ (修复后)
            → filterSignalsByStrategy(sigs, newStrategy) → 正确过滤 ✅
            → 构建新卡片DOM → 显示当前策略的信号 ✅
```
