# Context

源码：`packages/react-reconciler/src/ReactFiberNewContext.js`

## createContext 的数据结构

```javascript
const context = {
  $$typeof: REACT_CONTEXT_TYPE,
  _currentValue: defaultValue,   // 当前激活的值（主渲染器）
  _currentValue2: defaultValue,  // 副渲染器使用（React Native）
  Provider: context,             // 指向自身
  Consumer: { $$typeof: REACT_CONSUMER_TYPE, _context: context },
}
```

`Provider` 直接指向 context 自身，所以 `<Context.Provider>` 和 `<Context>` 是同一个东西。

---

## Provider：值的更新与栈管理

Provider 渲染时调用 `pushProvider`，把旧值压栈，更新 `_currentValue`：

```javascript
function pushProvider(providerFiber, context, nextValue) {
  push(valueCursor, context._currentValue, providerFiber)  // 旧值入栈
  context._currentValue = nextValue                         // 更新为新值
}
```

Provider 完成（completeWork）时调用 `popProvider`，从栈恢复旧值：

```javascript
function popProvider(context, providerFiber) {
  context._currentValue = valueCursor.current  // 恢复上一层的值
  pop(valueCursor, providerFiber)
}
```

**多层嵌套时的栈变化：**

```
<ThemeContext.Provider value="light">    pushProvider → stack: [null], _currentValue="light"
  <ThemeContext.Provider value="dark">   pushProvider → stack: [null,"light"], _currentValue="dark"
    <Consumer />                         readContext → 读到 "dark" ✓
  </ThemeContext.Provider>               popProvider  → _currentValue="light"
</ThemeContext.Provider>                 popProvider  → _currentValue=null
```

---

## Consumer：订阅依赖

`useContext(Context)` → `readContext(Context)`，每次调用都在 `fiber.dependencies` 上追加一个依赖节点：

```javascript
function readContextForConsumer(consumer, context) {
  const value = context._currentValue   // 读当前值

  const contextItem = {
    context,
    memoizedValue: value,   // 记录读取时的值，用于变化检测
    next: null,
  }

  // 追加到链表
  if (lastContextDependency === null) {
    consumer.dependencies = { lanes: NoLanes, firstContext: contextItem }
  } else {
    lastContextDependency = lastContextDependency.next = contextItem
  }
  return value
}
```

一个组件可以消费多个 Context，全部以链表形式挂在 `fiber.dependencies.firstContext`。

---

## Context 变化时如何触发重渲染

Provider 重新渲染时，比较新旧 value（`Object.is`），若变化则调用 `propagateContextChange`：

```javascript
// 遍历子树所有 fiber
while (fiber !== null) {
  const dependencies = fiber.dependencies
  if (dependencies !== null) {
    // 遍历该 fiber 的所有 context 依赖
    let dep = dependencies.firstContext
    while (dep !== null) {
      if (dep.context === changedContext) {
        // 命中！标记该 fiber 需要更新
        fiber.lanes = mergeLanes(fiber.lanes, renderLanes)
        // 向上标记祖先的 childLanes，防止被 bailout 跳过
        scheduleContextWorkOnParentPath(fiber.return, renderLanes)
        break
      }
      dep = dep.next
    }
  }
  fiber = nextFiber
}
```

不通过 setState，直接往 fiber.lanes 上打标记，下次渲染时该组件会被强制重新执行。

---

## 为什么 React.memo 拦不住 Context 变化

`React.memo` 的 bailout 检查分两步，**Context 检查在 props 比较之前**：

```javascript
function updateMemoComponent(...) {
  const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(current, renderLanes)

  if (!hasScheduledUpdateOrContext) {
    // props 浅比较
    if (compare(prevProps, nextProps)) {
      return bailoutOnAlreadyFinishedWork(...)  // 跳过渲染
    }
  }
  // 必须重新渲染
}

function checkScheduledUpdateOrContext(current, renderLanes) {
  if (includesSomeLane(current.lanes, renderLanes)) return true  // 有待处理更新
  if (checkIfContextChanged(current.dependencies)) return true   // Context 变化了！
  return false
}
```

`checkIfContextChanged` 对比 `memoizedValue` 和 `context._currentValue`，只要有一个 Context 值变了，就返回 true，props 比较直接被跳过，组件强制重渲染。

---

## 总结

```
createContext   → _currentValue 存当前值，Provider 指向自身

Provider 渲染   → pushProvider：旧值入栈，_currentValue = newValue
Provider 完成   → popProvider：_currentValue 恢复为上一层值

useContext      → readContext：读 _currentValue，在 fiber.dependencies 上挂依赖

Provider 值变化 → propagateContextChange：遍历子树，找依赖该 Context 的 fiber
                  → 直接给 fiber.lanes 打标记，强制重渲染
                  → React.memo 的 props 比较无法阻挡（Context 检查优先）
```
