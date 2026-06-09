# Transition

源码：`packages/react/src/ReactStartTransition.js`、`packages/react-reconciler/src/ReactFiberHooks.js`

## startTransition 做了什么

核心是设置一个全局 transition 上下文：

```javascript
function startTransition(scope) {
  const prevTransition = ReactSharedInternals.T  // 支持嵌套
  ReactSharedInternals.T = {}                    // 标记"当前在 transition 中"

  try {
    scope()   // 执行回调，里面的 setState 都能看到这个标记
  } finally {
    ReactSharedInternals.T = prevTransition  // 恢复
  }
}
```

`scope()` 执行期间，每次 `setState` 调用 `requestUpdateLane` 时，发现 `ReactSharedInternals.T` 不为空，就走 transition 分支，分配 TransitionLane。

---

## 同一个 startTransition 里的多个 setState 用同一个 Lane

`requestTransitionLane` 在同一次事件内只分配一次 lane，后续调用返回缓存：

```javascript
let currentEventTransitionLane = NoLane

function requestTransitionLane() {
  if (currentEventTransitionLane === NoLane) {
    currentEventTransitionLane = claimNextTransitionUpdateLane()  // 只分配一次
  }
  return currentEventTransitionLane  // 后续调用返回缓存
}
```

因此：

```javascript
startTransition(() => {
  setQuery('hello')    // → TransitionLane1
  setTheme('dark')     // → TransitionLane1（同一个）
})
```

两个 setState 用同一个 lane，在同一次渲染里处理，保证一致性。

---

## useTransition：isPending 的实现

`useTransition` 内部维护了一个 boolean state，通过**乐观更新**实现 isPending：

```javascript
function mountTransition() {
  const stateHook = mountStateImpl(false)   // isPending state，初始 false
  const start = startTransition.bind(null, fiber, stateHook.queue, true, false)
  //                                                              ↑     ↑
  //                                                        pending  finished
  return [false, start]
}
```

调用 `start(callback)` 时：

```javascript
// 1. 立即把 isPending 设为 true（SyncLane，同步提交）
dispatchOptimisticSetState(fiber, queue, pendingState = true)
//   Update { lane: SyncLane, revertLane: TransitionLane, action: true }

// 2. 执行 callback，里面的 setState 走 TransitionLane
scope()

// 3. 登记"完成后把 isPending 设为 false"（TransitionLane，延迟提交）
dispatchSetStateInternal(fiber, queue, finishedState = false, TransitionLane)
```

**时序：**

```
调用 start(callback)
  ↓
isPending = true（同步，用户立刻看到 loading 状态）
  ↓
transition 内容开始渲染（TransitionLane，低优先级）
  ↓
transition 渲染完成，提交
  ↓
isPending = false（用户看到新内容）
```

---

## useDeferredValue：被动的 transition

`useTransition` 是主动的——你控制哪些 setState 走低优先级。

`useDeferredValue` 是被动的——你告诉 React "这个值可以滞后"：

```javascript
const deferredQuery = useDeferredValue(query)
```

实现逻辑：

```javascript
function updateDeferredValueImpl(hook, prevValue, value) {
  if (is(value, prevValue)) return value  // 值没变，直接返回

  const shouldDeferValue = !includesOnlyNonUrgentLanes(renderLanes)
  //                        当前是紧急渲染（SyncLane 等），需要延迟

  if (shouldDeferValue) {
    // 紧急渲染时：返回旧值，同时安排一次低优先级渲染来更新
    const deferredLane = requestDeferredLane()  // DeferredLane，比 TransitionLane 还低
    currentlyRenderingFiber.lanes |= deferredLane
    return prevValue   // 本次渲染用旧值
  } else {
    // 低优先级渲染时：直接使用新值
    hook.memoizedState = value
    return value
  }
}
```

**效果：**

```
用户输入 → query 更新（SyncLane，紧急）
  ↓
输入框立刻响应
deferredQuery 还是旧值（本次紧急渲染里）
  ↓
安排一次 DeferredLane 渲染
  ↓
DeferredLane 渲染时，deferredQuery 更新为新值
搜索结果更新
```

---

## useTransition vs useDeferredValue

| | useTransition | useDeferredValue |
|--|--|--|
| 使用位置 | 事件处理器 | 组件渲染函数 |
| 控制方式 | 主动标记哪些 setState 是低优先级 | 被动声明某个值可以滞后 |
| isPending | 显式返回，可以展示 loading | 通过新旧值是否相同来判断 |
| Lane | TransitionLane | DeferredLane（更低） |
| 适合场景 | 有明确触发点的更新（按钮、输入） | 派生值、Props 驱动的渲染优化 |

两者本质相同：**把更新标记为低优先级，让紧急更新（用户输入）先响应**。

---

## 完整流程

```
startTransition(() => { setState(newValue) })
  │
  ├─ ReactSharedInternals.T = currentTransition  （标记 transition 上下文）
  │
  ├─ setState → requestUpdateLane
  │    → 发现 T 不为空 → requestTransitionLane
  │    → 分配 TransitionLane，缓存到 currentEventTransitionLane
  │
  ├─ 同一 startTransition 内其他 setState → 复用 currentEventTransitionLane
  │
  ├─ ReactSharedInternals.T = prevTransition  （恢复）
  │
  └─ scheduleUpdateOnFiber(TransitionLane)
        ↓
     Scheduler：NormalPriority 调度
        ↓
     workLoop：TransitionLane 的更新可被高优先级打断
        ↓
     Commit：提交 transition 结果，isPending → false
```
