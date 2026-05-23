# 买卖信号卡片与策略匹配卡片联动 - 技术方案文档

## 一、现状分析

### 1.1 信号数据结构与来源

当前信号数据通过两个途径提取：

**途径1：Flex items 提取** (`replaceHoldingsCard()`, 第323-337行)
```javascript
document.querySelectorAll('.flex.items-center.gap-2').forEach(function(it) {
  var spans = it.querySelectorAll(':scope > span');
  if (spans.length >= 4) {
    var type = spans[0].textContent.trim();  // "买入" 或 "卖出"
    // spans[1]=time, spans[2]=reason, spans[3]=price
    sigs.push({ type: type, time: time, reason: reason, price: price });
  }
});
```

**途径2：表格行提取** (`extractSignalsFromTable()`, 第97-125行)
```javascript
var cells = tr.querySelectorAll('td');
if (cells.length >= 4) {
  var type = cells[0].textContent.trim();  // "买入" 或 "卖出"
  // cells[1]=time, cells[2]=reason, cells[3]=price
  sigs.push({ type: type, time: time, reason: reason, price: price });
}
```

**关键发现**：
| 字段 | 来源 | 说明 |
|------|------|------|
| `type` | cells[0] / spans[0] | "买入" 或 "卖出" |
| `time` | cells[1] / spans[1] | 信号触发时间 |
| `reason` | cells[2] / spans[2] | 触发条件描述 |
| `price` | cells[3] / spans[3] | 信号价格 |
| **strategy** | **无** | **当前不提取策略名称** |

**结论**：当前信号数据结构**不包含策略名称字段**。如果信号表格存在第5列（策略名）或flex items存在第5个span，当前代码**完全忽略**。

### 1.2 当前策略选择器的实现方式

**现状**：`index.html` 内联脚本中**没有任何对策略匹配卡片的操作逻辑**。策略匹配卡片完全由React (`assets/index-97gBSAr5.js`) 渲染和管理。

从任务描述可知：
- 策略匹配卡片有一个下拉选择框
- 可选策略包括：均值回归、动量、趋势等
- 选择器可能是 `<select>` 元素或React自定义组件
- 卡片显示当前策略的匹配度（如"81%匹配度"）

**需要运行时探测**：
1. 策略选择器是原生 `<select>` 还是自定义div/span组件
2. 当前选中的策略值存储在 `value` 属性还是 `textContent` 中
3. 策略名称在DOM中的具体选择器路径

### 1.3 信号与策略的关联关系

| 关联维度 | 当前状态 | 分析 |
|----------|----------|------|
| 信号数据结构 | 无策略字段 | 需要增强提取逻辑 |
| 信号显示逻辑 | 显示所有信号最新一条 | `sigs[sigs.length - 1]` 无过滤 |
| 策略→信号链路 | 无链路 | 完全独立 |
| 卡片通信机制 | 无 | 需要新建 |

### 1.4 关键架构约束

1. **WeakSet不可删除**：`processedCards = new WeakSet()`，一旦卡片被标记为已处理，**无法移除**。这是联动刷新的最大技术障碍。
   ```javascript
   // replaceHoldingsCard() 第311行
   if (!hdCard || processedCards.has(hdCard)) return;
   // WeakSet 没有 delete 方法！
   ```

2. **React控制策略卡片**：策略匹配卡片由React管理，内联脚本只能通过DOM操作与其交互。

3. **ES5约束**：不能使用ES6+语法（箭头函数、const/let、解构等）。

4. **不可修改 `assets/index-97gBSAr5.js`**：所有逻辑必须在 `index.html` 内联脚本中实现。

---

## 二、方案选项

### 方案A：增强信号提取 + 按策略名过滤（推荐）

**核心思路**：
1. 增强信号提取逻辑，从DOM中读取策略名称（表格第5列 / flex第5个span）
2. 建立策略选择器监听机制
3. 策略变化时，直接更新买卖信号卡片内容（不经过replaceHoldingsCard的完整重建流程）

**架构设计**：
```
┌─────────────────────────────────────────────────────┐
│                    页面初始化                         │
│  replaceHoldingsCard() 首次创建买卖信号卡片            │
│  → 提取所有信号，存储到 window._allSignals            │
│  → 启动 listenStrategyChange() 监听策略变化           │
└─────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│              策略选择器变化检测层                     │
│  1. 查找策略匹配卡片的DOM节点                        │
│  2. 监听 <select> change 事件（如果是原生下拉框）     │
│  3. MutationObserver 兜底（如果是React自定义组件）    │
│  4. 定时轮询 作为最终兜底                            │
└─────────────────────────────────────────────────────┘
                           │
              策略变化事件  ▼
┌─────────────────────────────────────────────────────┐
│              信号过滤与更新层                         │
│  1. 读取当前选中策略名称                              │
│  2. 从 window._allSignals 中过滤匹配策略的信号         │
│  3. 调用 updateSignalCardDisplay() 直接更新DOM        │
│  4. 保持卡片等高同步                                 │
└─────────────────────────────────────────────────────┘
```

**关键实现细节**：

```javascript
// 新增全局状态（放在State区域，第46-50行之后）
var currentSelectedStrategy = '';
var allSignalsCache = [];

// 新增：增强信号提取，包含策略名称
function extractSignalsWithStrategy() {
  var sigs = [];
  // 途径1：Flex items（增强版）
  document.querySelectorAll('.flex.items-center.gap-2').forEach(function(it) {
    var spans = it.querySelectorAll(':scope > span');
    if (spans.length >= 4) {
      var type = spans[0].textContent.trim();
      if (type === '买入' || type === '卖出') {
        var sig = {
          type: type,
          time: spans[1] && spans[1].textContent.trim(),
          reason: spans[2] && spans[2].textContent.trim(),
          price: spans[3] && spans[3].textContent.trim(),
          strategy: spans[4] ? spans[4].textContent.trim() : ''  // 第5列=策略名
        };
        sigs.push(sig);
      }
    }
  });
  // 途径2：表格行（增强版）
  if (sigs.length === 0) {
    document.querySelectorAll('table tbody tr, table tr').forEach(function(tr) {
      var cells = tr.querySelectorAll('td');
      if (cells.length >= 4) {
        var type = cells[0].textContent.trim();
        if (type === '买入' || type === '卖出') {
          var sig = {
            type: type,
            time: cells[1] && cells[1].textContent.trim(),
            reason: cells[2] && cells[2].textContent.trim(),
            price: cells[3] && cells[3].textContent.trim(),
            strategy: cells[4] ? cells[4].textContent.trim() : ''  // 第5列=策略名
          };
          // 去重逻辑...
          sigs.push(sig);
        }
      }
    });
  }
  return sigs;
}

// 新增：获取当前选中的策略名称
function getSelectedStrategy() {
  // 方法1：查找策略匹配卡片中的 <select> 元素
  var allSelects = document.querySelectorAll('select');
  for (var i = 0; i < allSelects.length; i++) {
    var sel = allSelects[i];
    // 通过DOM位置判断：select是否在策略匹配卡片内
    var parent = sel.parentElement;
    while (parent && parent !== document.body) {
      if (parent.textContent.indexOf('策略匹配') >= 0 ||
          parent.textContent.indexOf('匹配度') >= 0) {
        return sel.value || sel.options[sel.selectedIndex].text;
      }
      parent = parent.parentElement;
    }
  }
  // 方法2：从策略匹配卡片的标题/标签读取
  var strategyCard = findStrategyMatchCard();
  if (strategyCard) {
    // 尝试读取显示的策略名（如"均值回归"）
    var strategyEls = strategyCard.querySelectorAll('h3, h4, .font-heading, [class*="strategy"], [data-strategy]');
    for (var j = 0; j < strategyEls.length; j++) {
      var text = strategyEls[j].textContent.trim();
      if (/^(均值回归|动量|趋势|突破|震荡|价值|成长)$/.test(text)) {
        return text;
      }
    }
  }
  return '';
}

// 新增：查找策略匹配卡片
function findStrategyMatchCard() {
  var node = findTextNode('策略匹配');
  if (node) return findParentCard(node);
  // 备选：通过匹配度文本查找
  var walker = document.createTreeWalker(document.body, 4, null, false);
  var n;
  while (n = walker.nextNode()) {
    if (/\d+%\s*匹配度/.test(n.textContent)) {
      return findParentCard(n);
    }
  }
  return null;
}

// 新增：过滤信号
function filterSignalsByStrategy(sigs, strategy) {
  if (!strategy) return sigs;  // 无策略选择，返回全部
  var filtered = [];
  for (var i = 0; i < sigs.length; i++) {
    // 精确匹配或包含匹配
    if (sigs[i].strategy === strategy ||
        sigs[i].reason.indexOf(strategy) >= 0) {
      filtered.push(sigs[i]);
    }
  }
  // 如果精确过滤无结果，返回全部（降级策略）
  return filtered.length > 0 ? filtered : sigs;
}

// 新增：更新买卖信号卡片显示（直接操作DOM，不经过replaceHoldingsCard）
function updateSignalCardDisplay() {
  // 找到买卖信号卡片
  var hdNode = findTextNode('买卖信号');
  if (!hdNode) return;
  var hdCard = findParentCard(hdNode);
  if (!hdCard) return;
  
  var strategy = getSelectedStrategy();
  currentSelectedStrategy = strategy;
  
  // 使用缓存的全部信号进行过滤
  var filtered = filterSignalsByStrategy(allSignalsCache, strategy);
  
  if (filtered.length > 0) {
    var s = filtered[filtered.length - 1]; // 最新一条
    // 更新DOM：信号类型、触发条件、价格等
    // ...（具体DOM操作，复用replaceHoldingsCard中的构建逻辑）
  }
}

// 新增：监听策略变化
function listenStrategyChange() {
  // 1. 事件监听（原生select）
  var strategyCard = findStrategyMatchCard();
  if (strategyCard) {
    var select = strategyCard.querySelector('select');
    if (select) {
      select.addEventListener('change', function() {
        setTimeout(updateSignalCardDisplay, 50);
      });
    }
  }
  
  // 2. MutationObserver监听（React自定义组件）
  if (strategyCard && !window._strategyObserverStarted) {
    var observer = new MutationObserver(function(mutations) {
      var newStrategy = getSelectedStrategy();
      if (newStrategy !== currentSelectedStrategy) {
        currentSelectedStrategy = newStrategy;
        updateSignalCardDisplay();
      }
    });
    observer.observe(strategyCard, {
      childList: true, subtree: true, characterData: true
    });
    window._strategyObserverStarted = true;
  }
}
```

**优点**：
- 不修改 `replaceHoldingsCard()` 核心流程，风险低
- 直接操作DOM更新，无WeakSet限制问题
- 信号缓存机制避免重复DOM遍历
- 多重策略检测（精确匹配 + reason字段模糊匹配）

**缺点**：
- 需要维护两套信号显示逻辑（首次创建 + 后续更新）
- 依赖于信号DOM中确实包含策略名称信息
- 如果策略选择器是复杂的React组件，MutationObserver可能过于敏感

**风险**：
- `updateSignalCardDisplay()` 中的DOM操作需要与 `replaceHoldingsCard()` 保持结构一致
- 信号缓存 `allSignalsCache` 可能在页面数据刷新后过期

---

### 方案B：联动刷新（信号重新生成）

**核心思路**：
1. 监听策略选择器变化
2. 变化时绕过 `processedCards` 的WeakSet检查
3. 触发 `replaceHoldingsCard()` 重新执行完整流程
4. 重新从DOM提取信号（假设React已更新了信号数据）

**WeakSet绕过机制**：
```javascript
// 新增全局标志
var forceRefreshSignalCard = false;

// 修改 replaceHoldingsCard() 的入口检查
function replaceHoldingsCard() {
  var hdNode = findTextNode('持仓管理');  // 或'买卖信号'
  // ...
  // 第311行修改：允许强制刷新
  if (!hdCard || (processedCards.has(hdCard) && !forceRefreshSignalCard)) return;
  forceRefreshSignalCard = false;
  // ... 其余逻辑不变
}

// 策略变化时
function onStrategyChange() {
  forceRefreshSignalCard = true;
  // 清除买卖信号卡片的innerHTML，让它重新渲染
  var hdNode = findTextNode('买卖信号');
  if (hdNode) {
    var hdCard = findParentCard(hdNode);
    if (hdCard) {
      hdCard.innerHTML = '';  // 清空，让replaceHoldingsCard重新构建
    }
  }
  replaceHoldingsCard();  // 重新执行
}
```

**优点**：
- 复用现有 `replaceHoldingsCard()` 的全部逻辑，无需重复实现
- 信号总是从DOM最新提取，不会使用过期缓存
- 代码改动量少

**缺点**：
- **关键风险**：清空innerHTML会导致卡片闪烁
- 策略变化时不一定伴随信号DOM的更新（React可能只是改变策略匹配卡片自身）
- 如果React没有更新信号数据，重新提取的信号仍然是全部策略的
- 频繁的策略切换会导致频繁的卡片重建

**风险**：
- 高：卡片重建导致用户体验闪烁
- 中：假设"策略切换→React更新信号DOM"可能不成立
- 低：innerHTML清空后若replaceHoldingsCard执行失败，卡片会空白

---

### 方案C：MutationObserver状态同步 + 响应式更新

**核心思路**：
1. 建立一个中央状态对象 `window._signalState`
2. 大范围MutationObserver监听策略卡片的变化
3. 变化时解析出新策略，更新状态
4. 状态变化触发自定义事件，买卖信号卡片响应更新

**实现**：
```javascript
// 中央状态
window._signalState = {
  currentStrategy: '',
  allSignals: [],
  filteredSignals: []
};

// 大范围Observer
var stateObserver = new MutationObserver(function(mutations) {
  mutations.forEach(function(mutation) {
    // 检测策略变化...
    var newStrategy = detectStrategyChange(mutation);
    if (newStrategy !== window._signalState.currentStrategy) {
      window._signalState.currentStrategy = newStrategy;
      // 触发自定义事件
      var event = document.createEvent('CustomEvent');
      event.initCustomEvent('strategyChanged', true, true, {
        strategy: newStrategy
      });
      document.dispatchEvent(event);
    }
  });
});

// 买卖信号卡片监听
document.addEventListener('strategyChanged', function(e) {
  updateSignalDisplay(e.detail.strategy);
});
```

**优点**：
- 架构清晰，状态与视图分离
- 可扩展性强，未来可添加更多联动

**缺点**：
- ES5环境下代码冗长（没有CustomEvent的便捷API）
- 过度设计，当前只有两个组件需要联动
- MutationObserver范围过大，性能开销高
- 在已有debounced MutationObserver基础上再嵌套一层，增加复杂度

**风险**：
- 自定义事件在旧浏览器兼容性
- 状态对象与DOM可能不一致
- 调试困难

---

## 三、推荐方案

### 推荐：方案A（增强信号提取 + 按策略名过滤）

**推荐理由**：

| 评估维度 | 方案A | 方案B | 方案C |
|----------|-------|-------|-------|
| 代码侵入性 | 低（新增函数为主） | 中（修改核心函数） | 高（重构架构） |
| WeakSet兼容性 | 完全兼容 | 需要绕过机制 | 完全兼容 |
| 闪烁问题 | 无 | 有（innerHTML重建） | 无 |
| 性能影响 | 低（缓存+局部更新） | 中（完整重建） | 高（大范围Observer） |
| 可维护性 | 高 | 中 | 低 |
| 风险等级 | 低 | 中 | 高 |

### 实施步骤

#### Step 1: 增强信号提取（修改两个函数）

**文件**: `index.html` 内联脚本

**修改1**: `extractSignalsFromTable()` 第97-125行
- 在 `sigs.push({...})` 前增加策略字段读取
- 尝试读取 `cells[4]` 作为策略名称

**修改2**: `replaceHoldingsCard()` 第323-337行的flex提取逻辑
- 在 `sigs.push({...})` 前增加策略字段读取
- 尝试读取 `spans[4]` 作为策略名称

#### Step 2: 新增全局状态变量（State区域）

**位置**: 第46-50行之后
```javascript
// ===== Strategy-Signal Linkage State =====
var currentSelectedStrategy = '';
var allSignalsCache = [];
var strategyObserverStarted = false;
```

#### Step 3: 新增辅助函数（Helpers区域之后）

**位置**: 第141行之后
新增以下函数：
1. `findStrategyMatchCard()` - 查找策略匹配卡片
2. `getSelectedStrategy()` - 获取当前选中策略
3. `filterSignalsByStrategy(sigs, strategy)` - 按策略过滤信号
4. `updateSignalCardDisplay()` - 更新信号卡片显示
5. `listenStrategyChange()` - 监听策略变化

#### Step 4: 修改 replaceHoldingsCard() 的信号缓存

**位置**: 第338-343行（flex提取后）
- 在提取完信号后，将 `allSignalsCache = sigs`

#### Step 5: 修改 init() 启动策略监听

**位置**: 第749-763行
- 在 `runAll()` 调用后增加 `listenStrategyChange()` 启动

#### Step 6: 修改 runAll() 添加策略监听启动

**位置**: 第690-696行
- 在 `replaceHoldingsCard()` 调用后增加策略监听启动

### 关键代码修改清单

| 修改类型 | 目标位置 | 行号范围 | 说明 |
|----------|----------|----------|------|
| 修改 | `extractSignalsFromTable()` | 114-119 | 增加strategy字段读取 |
| 修改 | `replaceHoldingsCard()` flex提取 | 329-334 | 增加strategy字段读取 |
| 修改 | `replaceHoldingsCard()` 缓存信号 | 338-343 | 增加`allSignalsCache = sigs` |
| 新增 | State区域 | 50-53 | 3个全局变量 |
| 新增 | Helpers之后 | 141+ | 5个新函数 |
| 修改 | `runAll()` | 693-694 | 启动策略监听 |

### 不需要修改的部分

| 部分 | 理由 |
|------|------|
| `processedCards` WeakSet机制 | 方案A直接操作DOM更新，不依赖重新走replaceHoldingsCard |
| `syncCardHeight()` | 等高同步保持现有逻辑 |
| `removeStarCards()` 系列 | 星号卡片移除与策略联动无关 |
| `setupReflectionFold()` | 信号反思折叠与策略联动无关 |
| `enhanceRealtimeCard()` | 实时行情增强与策略联动无关 |
| MutationObserver主监听 | 现有100ms debounce机制足够 |
| 轮询机制 (`startPoll`) | 策略监听使用独立Observer+事件 |
| `assets/index-97gBSAr5.js` | 约束要求不修改 |

### 风险评估

| 风险项 | 等级 | 缓解措施 |
|--------|------|----------|
| 信号DOM中无策略名称列 | **高** | 降级为显示全部信号；同时尝试从reason字段模糊匹配 |
| 策略选择器无法通过DOM定位 | 中 | 多重检测机制（select查找 → 文本匹配 → MutationObserver） |
| updateSignalCardDisplay() DOM结构与replaceHoldingsCard()不一致 | 中 | 代码审查确保结构一致；提取公共构建函数 |
| 频繁策略切换导致性能问题 | 低 | update函数轻量级（只更新文本，不重建结构） |
| 信号缓存过期 | 低 | 页面数据刷新时（route change）清空缓存 |

---

## 四、代码修改详情（伪代码/实现参考）

### 4.1 新增全局状态（第50行后插入）

```javascript
  // ===== Strategy-Signal Linkage State =====
  var currentSelectedStrategy = '';
  var allSignalsCache = [];
  var strategyObserverStarted = false;
```

### 4.2 修改 extractSignalsFromTable()（第114-119行替换）

```javascript
          if (!isDup) {
            var strategyName = cells[4] ? cells[4].textContent.trim() : '';
            sigs.push({
              type: type,
              time: time,
              reason: cells[2] && cells[2].textContent.trim(),
              price: cells[3] && cells[3].textContent.trim(),
              strategy: strategyName
            });
          }
```

### 4.3 修改 replaceHoldingsCard() flex提取（第329-334行替换）

```javascript
          var strategyName = spans[4] ? spans[4].textContent.trim() : '';
          sigs.push({
            type: type,
            time: spans[1] && spans[1].textContent.trim(),
            reason: spans[2] && spans[2].textContent.trim(),
            price: spans[3] && spans[3].textContent.trim(),
            strategy: strategyName
          });
```

### 4.4 新增信号缓存（第342行后插入）

```javascript
    // Cache all signals for strategy filtering
    allSignalsCache = sigs;
```

### 4.5 新增辅助函数（第141行后插入全部）

```javascript
  // ===== Strategy Match Card Detection =====
  function findStrategyMatchCard() {
    var node = findTextNode('策略匹配');
    if (node) return findParentCard(node);
    var walker = document.createTreeWalker(document.body, 4, null, false);
    var n;
    while (n = walker.nextNode()) {
      if (/\d+%\s*匹配度/.test(n.textContent)) {
        return findParentCard(n);
      }
    }
    return null;
  }

  function getSelectedStrategy() {
    var strategyCard = findStrategyMatchCard();
    if (!strategyCard) return '';
    var select = strategyCard.querySelector('select');
    if (select) {
      return select.value || (select.options[select.selectedIndex] && select.options[select.selectedIndex].text) || '';
    }
    var headingEls = strategyCard.querySelectorAll('h3, h4, .font-heading, [class*="strategy"]');
    for (var i = 0; i < headingEls.length; i++) {
      var text = headingEls[i].textContent.trim();
      if (/^(均值回归|动量|趋势|突破|震荡|价值|成长|反转|波动率)$/.test(text)) return text;
    }
    return '';
  }

  function filterSignalsByStrategy(sigs, strategy) {
    if (!strategy || !sigs || sigs.length === 0) return sigs;
    var filtered = [];
    for (var i = 0; i < sigs.length; i++) {
      if (sigs[i].strategy && sigs[i].strategy === strategy) {
        filtered.push(sigs[i]);
      } else if (sigs[i].reason && sigs[i].reason.indexOf(strategy) >= 0) {
        filtered.push(sigs[i]);
      }
    }
    return filtered.length > 0 ? filtered : sigs;
  }

  function updateSignalCardDisplay() {
    var strategy = getSelectedStrategy();
    if (strategy === currentSelectedStrategy && currentSelectedStrategy !== '') return;
    currentSelectedStrategy = strategy;

    var hdNode = findTextNode('买卖信号');
    if (!hdNode) return;
    var hdCard = findParentCard(hdNode);
    if (!hdCard) return;

    var filtered = filterSignalsByStrategy(allSignalsCache, strategy);
    var s = filtered.length > 0 ? filtered[filtered.length - 1] : null;

    // 更新信号类型badge
    var badgeEl = hdCard.querySelector('.font-heading.text-2xl');
    if (badgeEl && s) {
      badgeEl.textContent = s.type;
      var sigColor = s.type === '买入' ? '#00FF94' : '#FF2A6D';
      var sigBg = s.type === '买入' ? 'rgba(0,255,148,0.08)' : 'rgba(255,42,109,0.08)';
      var sigBrdr = s.type === '买入' ? 'rgba(0,255,148,0.20)' : 'rgba(255,42,109,0.20)';
      badgeEl.style.color = sigColor;
      badgeEl.style.background = sigBg;
      badgeEl.style.borderColor = sigBrdr;
    }

    // 更新触发条件
    if (s && s.reason) {
      var triggerSpans = hdCard.querySelectorAll('.font-mono.text-xs.text-pure');
      for (var i = 0; i < triggerSpans.length; i++) {
        if (triggerSpans[i].previousElementSibling &&
            triggerSpans[i].previousElementSibling.innerHTML.indexOf('&#9679;') >= 0 &&
            triggerSpans[i].previousElementSibling.style.color === 'rgb(0, 255, 148)') {
          triggerSpans[i].textContent = s.reason;
          break;
        }
      }
    }

    // 更新信号价
    if (s && s.price) {
      var priceLabels = hdCard.querySelectorAll('.font-mono');
      for (var j = 0; j < priceLabels.length; j++) {
        if (priceLabels[j].textContent === '信号价') {
          var priceVal = priceLabels[j].nextElementSibling;
          if (priceVal) priceVal.textContent = s.price;
          break;
        }
      }
    }

    // 更新时间
    if (s && s.time) {
      var timeLabels = hdCard.querySelectorAll('.font-mono');
      for (var k = 0; k < timeLabels.length; k++) {
        if (timeLabels[k].textContent === '时间') {
          var timeVal = timeLabels[k].nextElementSibling;
          if (timeVal) timeVal.textContent = s.time;
          break;
        }
      }
    }

    // 更新历史信号数量
    var countEls = hdCard.querySelectorAll('.font-mono.text-\[9px\].text-ash\/25');
    for (var m = 0; m < countEls.length; m++) {
      if (countEls[m].textContent.indexOf('历史信号') >= 0) {
        countEls[m].textContent = (strategy ? '当前策略' : '历史') + '信号共 ' + filtered.length + ' 条';
        break;
      }
    }
  }

  function listenStrategyChange() {
    if (strategyObserverStarted) return;
    strategyObserverStarted = true;

    var strategyCard = findStrategyMatchCard();
    if (!strategyCard) return;

    var select = strategyCard.querySelector('select');
    if (select) {
      select.addEventListener('change', function() {
        setTimeout(updateSignalCardDisplay, 80);
      });
    }

    var observer = new MutationObserver(function() {
      clearTimeout(window._strategyDebounceTimer);
      window._strategyDebounceTimer = setTimeout(updateSignalCardDisplay, 150);
    });
    observer.observe(strategyCard, { childList: true, subtree: true, characterData: true });
  }
```

### 4.6 修改 runAll()（第693行后插入）

```javascript
    // 启动策略-信号联动监听
    if (!strategyObserverStarted) {
      listenStrategyChange();
    }
```

---

## 五、Fallback策略

### 情况1：信号DOM中没有策略名称列

如果 `cells[4]` 和 `spans[4]` 都不存在或不是策略名称：
- `filterSignalsByStrategy()` 会退化为返回全部信号
- 联动效果降级为"策略选择不改变信号显示"
- **需要进一步调查**：从"策略信号 (22个) — 均值回归"这类表格标题中提取策略名称，按表格区域归属给信号

### 情况2：策略选择器完全无法通过DOM定位

如果 `findStrategyMatchCard()` 返回null：
- 联动功能静默失效
- 买卖信号卡片保持当前行为（显示全部信号最新一条）
- 不影响任何现有功能

### 情况3：updateSignalCardDisplay() DOM选择器不匹配

如果卡片更新失败：
- 首次加载的信号显示正常（由 `replaceHoldingsCard()` 保证）
- 策略切换时信号不更新，但不破坏现有显示
- 可通过增加更robust的DOM选择器来改善

---

## 六、验证检查清单

实施完成后，按以下清单验证：

- [ ] 页面首次加载时，买卖信号卡片正常显示
- [ ] 切换策略选择器时，买卖信号卡片内容相应更新
- [ ] 切换到一个没有信号的策略时，显示"当前无活跃买卖信号"
- [ ] 卡片等高同步仍然正常工作
- [ ] 星号卡片移除仍然正常工作
- [ ] 信号反思折叠仍然正常工作
- [ ] 实时行情增强仍然正常工作
- [ ] 路由切换后功能仍然正常
- [ ] 无JavaScript报错

---

*文档版本: 1.0*
*分析基于: index.html (行数768)*
*约束: ES5, 仅修改index.html内联脚本*
