# 代码审计报告：`index.html` 联动代码

## 1. 函数调用关系图

```
用户点击策略按钮
  └── React 更新 DOM
       └── MutationObserver 回调 (startStrategyLinkage 中注册)
            ├── [F1] findStrategyMatchCard()  .............. 策略卡片可能未渲染
            │     └── findTextNode('策略匹配') .............. 文本节点不存在
            │           └── findParentCard(node) ............ node为null
            └── readCurrentStrategy()
                  ├── 方法1: querySelector('select') ........ [F2] select值不在strategyNames中
                  │     └── return 策略名
                  ├── 方法2: querySelectorAll('button') ..... [F3] 按钮文本含额外字符/图标
                  │     └── return 策略名
                  ├── 方法3: heading 元素
                  └── 方法4: 所有子元素
       └── dispatchStrategyEvent(strategyName)
             ├── [F4] document.createEvent 不存在时 .......... else分支创建普通对象
             │     └── event = { type: ..., detail: ... }    非Event实例，dispatchEvent报错
             └── document.dispatchEvent(event) .............. [F5] ES5兼容性问题
       └── CustomEvent: strategyChanged
            └── onStrategyChangeEvent(event)
                  ├── event.detail.strategy .................. [F5] 可能undefined
                  ├── 重复策略检查 (newStrategy === current)
                  │     └── 相同则return (正确行为)
                  ├── querySelectorAll('.data-card') ......... [F6] 找不到买卖信号卡片
                  │     └── 查找包含 '买卖信号' 的卡片
                  ├── processedCards.delete(hdCard) .......... 正常(OK) - Object有delete方法
                  ├── hdCard.innerHTML = '' .................... ★ BUG #1: 删除 '买卖信号' 文本!
                  └── replaceHoldingsCard()
                        ├── findTextNode('持仓管理') ........... 返回null (已被替换)
                        ├── findTextNode('买卖信号') ........... 返回null (刚被innerHTML=''删除!)
                        │                                      └── ★ return; (函数提前退出!)
                        ├── processedCards.has(hdCard) ......... 永不到达
                        ├── extractSignalsFromTable() .......... 永不到达
                        ├── filterSignalsByStrategy() .......... 永不到达
                        └── 构建卡片DOM ........................ 永不到达 (BUG!)
```

---

## 2. 完整失败点分析表

### 2.1 致命Bug (Critical)

| # | 失败点 | 代码位置 | 失败条件 | 结果 |
|---|--------|---------|---------|------|
| **C1** | **onStrategyChangeEvent 清空 innerHTML 后 replaceHoldingsCard 无法找到卡片** | `hdCard.innerHTML = ''` (line 332) → `replaceHoldingsCard()` (line 334) | `hdCard.innerHTML = ''` 删除了 '买卖信号' 文本节点，随后 `replaceHoldingsCard()` 中 `findTextNode('买卖信号')` 返回 null | **策略切换后买卖信号卡片被永久清空，永远无法重建** |
| C2 | dispatchStrategyEvent else 分支创建非 Event 对象 | line 215-216 | 当 `document.createEvent` 不存在时，else 分支创建普通对象 `{type, detail}`，然后调用 `document.dispatchEvent(event)` | `TypeError: Argument 1 of EventTarget.dispatchEvent is not an object.` |

### 2.2 高严重度 (High)

| # | 失败点 | 代码位置 | 失败条件 | 结果 |
|---|--------|---------|---------|------|
| H1 | MutationObserver 只附加一次 | line 349-360 | `findStrategyMatchCard()` 在 `startStrategyLinkage()` 调用时只执行一次。如果策略卡片尚未渲染（React 异步渲染），observer 不会被附加 | 策略卡片 DOM 变化无法被检测，策略联动完全失效 |
| H2 | MutationObserver 引用的是可能过期的 DOM 元素 | line 349 | `strategyCard` 是 React 管理的 DOM，如果 React 完全重建策略卡片（非子元素更新），observer 丢失 | 策略变化检测失效 |
| H3 | processedCards 在路由切换时完全清空，但 MutationObserver 和事件监听未重置 | line 978 | `processedCards._cards = []` 清空所有引用，但 `startStrategyLinkage()` 中 `strategyObserving` 标志阻止重新设置 observer | 路由切换后策略 observer 可能重复注册或丢失 |

### 2.3 中严重度 (Medium)

| # | 失败点 | 代码位置 | 失败条件 | 结果 |
|---|--------|---------|---------|------|
| M1 | readCurrentStrategy 的严格相等匹配 | line 186 | 按钮文本如 `"均值回归 "`（trim后ok）或 `"📊 均值回归"`（trim后仍有图标）与策略名列表严格相等失败 | 无法读取当前策略，返回空字符串 |
| M2 | readCurrentStrategy select 值匹配 | line 177 | select.value 可能包含前缀后缀如 `"strategy-均值回归"` 或 `"均值回归 (推荐)"` | `indexOf` 检查通过，但可能匹配错误策略 |
| M3 | findTextNode 严格匹配 | line 70 | 文本节点内容如 `"买卖信号\n"` 或 `" 买卖信号 "` 在 trim 后能匹配，但如果有多余字符则不匹配 | 找不到目标卡片 |
| M4 | extractSignalsFromTable 重复检查逻辑过简 | line 122-124 | 仅用 `type` 和 `time` 判断重复，两个不同 `price` 或 `reason` 但同 type+time 的信号会被视为重复 | 信号丢失 |
| M5 | startStrategyLinkage 轮询只检查 location.hash | line 365 | `if (!location.hash.includes('monitor')) return;` 只在非 monitor 页面返回。如果在 monitor 页面但信号卡片被 React 重建，轮询只调用 `removeStarCards()` | 轮询不会触发策略重新检测（只在 MutationObserver 失效时备用） |

### 2.4 低严重度/代码质量问题 (Low)

| # | 问题 | 代码位置 | 说明 |
|---|------|---------|------|
| L1 | refreshSignalCardDOM 死代码 | line 235-308 | 函数定义但从未被调用（注释说使用 FULL REBUILD approach） |
| L2 | extractSignalsFromTable 中 `:scope` 选择器兼容性 | line 573 | `it.querySelectorAll(':scope > span')` 在 IE 中不支持 |
| L3 | dispatchStrategyEvent 使用已废弃 API | line 211-213 | `document.createEvent` + `initCustomEvent` 已被废弃，应使用 `new CustomEvent()` |
| L4 | onRouteChange 中多次调用 runAll | line 981-983 | 使用多个 setTimeout 而非更可靠的 Promise/回调机制 |
| L5 | 轮询和 MutationObserver 可能同时触发 | - | 两者可能同时检测到策略变化，导致 replaceHoldingsCard 被调用两次 |

---

## 3. 根因分析 (Top 3)

### 🥇 Root Cause #1: onStrategyChangeEvent 中 innerHTML='' 破坏了卡片查找锚点

**问题代码** (line 329-335):
```javascript
if (hdCard) {
    processedCards.delete(hdCard);
    hdCard.innerHTML = '';       // ← 删除了 '买卖信号' 文本!
    replaceHoldingsCard();        // ← 调用 findTextNode('买卖信号') → 永远返回 null!
}
```

**原因链**:
1. `replaceHoldingsCard()` 依赖 `findTextNode('持仓管理')` 或 `findTextNode('买卖信号')` 来定位目标卡片
2. 第一次运行时，通过 '持仓管理' 找到卡片，将其内容替换为包含 '买卖信号' 标题的新内容
3. 策略切换时，`onStrategyChangeEvent` 将 `hdCard.innerHTML = ''`，删除了 '买卖信号' 文本
4. 随后调用 `replaceHoldingsCard()`，`findTextNode('持仓管理')` 返回 null（已被替换），`findTextNode('买卖信号')` 也返回 null（刚被删除）
5. 函数在第 556 行 `if (!hdNode) return;` 提前退出，卡片永远保持空白

**修复方案**:
```javascript
// === 修改 replaceHoldingsCard 函数签名，支持传入目标卡片 ===
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
    // ... 其余代码不变
}

// === 修改 onStrategyChangeEvent ===
function onStrategyChangeEvent(event) {
    var newStrategy = '';
    if (event.detail && event.detail.strategy) {
        newStrategy = event.detail.strategy;
    }
    if (newStrategy === window._signalState.currentStrategy) return;
    window._signalState.currentStrategy = newStrategy;

    var hdCard = window._signalState.hdCardRef;  // 使用已保存的引用!
    if (hdCard) {
        processedCards.delete(hdCard);
        hdCard.innerHTML = '';
        replaceHoldingsCard(hdCard);  // 传入目标卡片!
    }
}
```

---

### 🥈 Root Cause #2: startStrategyLinkage 的 MutationObserver 生命周期管理缺陷

**问题代码** (line 339-371):
```javascript
function startStrategyLinkage() {
    var state = window._signalState;
    if (state.strategyObserving) return;  // ← 只执行一次!
    state.strategyObserving = true;

    var strategyCard = findStrategyMatchCard();  // ← 只查找一次!
    if (strategyCard) {
        var observer = new MutationObserver(...);
        observer.observe(strategyCard, ...);  // ← observer 引用的是可能过期的 DOM!
    }
    // ...
}
```

**原因链**:
1. `strategyObserving` 标志确保函数只执行一次
2. `findStrategyMatchCard()` 在调用时只执行一次查找
3. 如果 React 尚未渲染策略卡片，`strategyCard` 为 null，observer 不会被附加
4. 即使成功附加，如果 React 重建策略卡片（替换整个元素），observer 引用的是已不在 DOM 中的过期元素
5. 路由切换时 `onRouteChange` 清空 `processedCards._cards` 但不重置 `strategyObserving`，导致无法重新初始化

**修复方案**:
```javascript
function startStrategyLinkage() {
    var state = window._signalState;
    if (state.strategyObserving) {
        // 检查是否需要重新附加 observer（DOM 可能被 React 重建）
        reattachStrategyObserver();
        return;
    }
    state.strategyObserving = true;
    attachStrategyObserver();
}

// 将 observer 附加逻辑提取为独立函数
var strategyObserver = null;
var strategyObservedElement = null;

function attachStrategyObserver() {
    var strategyCard = findStrategyMatchCard();
    if (!strategyCard) return false;
    
    if (strategyObserver) {
        strategyObserver.disconnect();
    }
    
    strategyObserver = new MutationObserver(function() {
        clearTimeout(strategyDebounceTimer);
        strategyDebounceTimer = setTimeout(function() {
            var newStrategy = readCurrentStrategy();
            if (newStrategy && newStrategy !== window._signalState.currentStrategy) {
                dispatchStrategyEvent(newStrategy);
            }
        }, 150);
    });
    strategyObserver.observe(strategyCard, { childList: true, subtree: true, characterData: true });
    strategyObservedElement = strategyCard;
    return true;
}

function reattachStrategyObserver() {
    // 检查被观察的元素是否仍在 DOM 中
    if (strategyObservedElement && !document.contains(strategyObservedElement)) {
        attachStrategyObserver();
    }
}

// 修改 onRouteChange 中的重置逻辑
function onRouteChange() {
    if (location.href !== lastUrl) {
        lastUrl = location.href;
        if (location.hash.includes('monitor')) {
            processedCards._cards = [];
            window._signalState.hdCardRef = null;
            window._signalState.strategyObserving = false;  // ← 重置标志
            window._signalState.currentStrategy = '';        // ← 重置策略
            attachStrategyObserver();                        // ← 重新附加 observer
            setTimeout(runAll, 500);
            setTimeout(runAll, 1200);
            setTimeout(runAll, 2500);
            startPoll();
        } else {
            stopPoll();
        }
    }
}
```

---

### 🥉 Root Cause #3: dispatchStrategyEvent 的降级分支会抛出异常

**问题代码** (line 209-218):
```javascript
function dispatchStrategyEvent(strategyName) {
    var event;
    if (document.createEvent) {
        event = document.createEvent('CustomEvent');
        event.initCustomEvent('strategyChanged', true, true, { strategy: strategyName });
    } else {
        event = { type: 'strategyChanged', detail: { strategy: strategyName } };  // ← 普通对象!
    }
    document.dispatchEvent(event);  // ← else分支: TypeError!
}
```

**原因链**:
1. 代码意图是提供 ES5 兼容性降级
2. 当 `document.createEvent` 不存在时（现代浏览器不可能，但测试环境/某些 polyfill 场景可能），else 分支创建一个普通对象
3. `document.dispatchEvent()` 要求参数必须是 `Event` 接口的实例
4. 传入普通对象会抛出 `TypeError`，导致策略变化事件无法分发

**修复方案**:
```javascript
function dispatchStrategyEvent(strategyName) {
    var event;
    if (typeof CustomEvent === 'function') {
        // 现代浏览器标准方式
        event = new CustomEvent('strategyChanged', {
            bubbles: true,
            cancelable: true,
            detail: { strategy: strategyName }
        });
    } else if (document.createEvent) {
        // ES5 兼容降级
        event = document.createEvent('CustomEvent');
        event.initCustomEvent('strategyChanged', true, true, { strategy: strategyName });
    } else {
        // 最低兼容: 创建最小 Event 对象
        event = document.createEvent('Event');
        event.initEvent('strategyChanged', true, true);
        event.detail = { strategy: strategyName };
    }
    document.dispatchEvent(event);
}
```

---

## 4. 完整修复后的关键函数

### 4.1 修改后的 replaceHoldingsCard (line 552)

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
    // ... 其余代码完全不变
}
```

### 4.2 修改后的 onStrategyChangeEvent (line 311)

```javascript
function onStrategyChangeEvent(event) {
    var newStrategy = '';
    if (event.detail && event.detail.strategy) {
        newStrategy = event.detail.strategy;
    }
    if (newStrategy === window._signalState.currentStrategy) return;
    window._signalState.currentStrategy = newStrategy;

    // 使用已保存的引用直接定位卡片，不再依赖文本搜索
    var hdCard = window._signalState.hdCardRef;
    if (hdCard) {
        processedCards.delete(hdCard);
        hdCard.innerHTML = '';
        replaceHoldingsCard(hdCard);  // 传入目标卡片引用!
    }
}
```

### 4.3 修改后的 dispatchStrategyEvent (line 209)

```javascript
function dispatchStrategyEvent(strategyName) {
    var event;
    if (typeof CustomEvent === 'function') {
        event = new CustomEvent('strategyChanged', {
            bubbles: true,
            cancelable: true,
            detail: { strategy: strategyName }
        });
    } else if (document.createEvent) {
        event = document.createEvent('CustomEvent');
        event.initCustomEvent('strategyChanged', true, true, { strategy: strategyName });
    } else {
        event = document.createEvent('Event');
        event.initEvent('strategyChanged', true, true);
        event.detail = { strategy: strategyName };
    }
    document.dispatchEvent(event);
}
```

### 4.4 路由切换重置逻辑 (line 978)

```javascript
function onRouteChange() {
    if (location.href !== lastUrl) {
        lastUrl = location.href;
        if (location.hash.includes('monitor')) {
            processedCards._cards = [];
            window._signalState.hdCardRef = null;
            window._signalState.currentStrategy = '';
            window._signalState.strategyObserving = false;  // 重置以允许重新初始化
            setTimeout(runAll, 500);
            setTimeout(runAll, 1200);
            setTimeout(runAll, 2500);
            startPoll();
        } else {
            stopPoll();
        }
    }
}
```

---

## 5. 数据流验证

修复后的策略切换数据流:

```
用户点击策略按钮
  → React 更新策略卡片 DOM
    → MutationObserver 检测到变化
      → debounce 150ms 后执行
        → readCurrentStrategy() 读取新策略名
          → dispatchStrategyEvent('均值回归')  [CustomEvent]
            → onStrategyChangeEvent(event)
              → event.detail.strategy === '均值回归' ✓
              → newStrategy !== window._signalState.currentStrategy ✓
              → window._signalState.currentStrategy = '均值回归'
              → hdCard = window._signalState.hdCardRef  (直接引用) ✓
              → processedCards.delete(hdCard)  ✓ (Object.delete 正常工作)
              → hdCard.innerHTML = ''  (清空旧内容)
              → replaceHoldingsCard(hdCard)  (传入卡片引用!)
                → targetCard 存在，跳过 findTextNode 查找 ✓
                → processedCards.has(hdCard)? 刚delete，返回false ✓
                → extractSignalsFromTable() 提取信号 ✓
                → filterSignalsByStrategy(sigs, '均值回归') 过滤 ✓
                → 构建新卡片 DOM ✓
                → processedCards.add(hdCard) ✓
                → 完成!
```

---

## 6. 测试建议

1. **测试 Root Cause #1**: 
   - 打开 monitor 页面，确认买卖信号卡片正常显示
   - 切换策略，观察卡片是否更新（修复前卡片变空白）

2. **测试 Root Cause #2**:
   - 切换到其他页面再切回 monitor，确认策略联动仍然工作
   - 快速切换策略多次，确认 observer 正常工作

3. **测试 Root Cause #3**:
   - 在测试环境中模拟 `document.createEvent = null`，确认事件分发不报错

---

*审计完成时间：基于代码行 1-1032 的完整分析*
