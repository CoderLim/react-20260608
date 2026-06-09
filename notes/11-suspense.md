# Suspense

源码：`packages/react-reconciler/src/ReactFiberThrow.js`、`ReactFiberBeginWork.js`

## 核心机制

Suspense 的本质是**用异常机制中断渲染**：子组件抛出一个 Promise，React 捕获后展示 fallback，Promise resolve 后重新渲染。

---

## 第一步：子组件抛出 Promise

通过 `use(promise)` 触发（旧版是直接 throw promise）：

```javascript
function use(usable) {
  if (typeof usable.then === 'function') {
    return useThenable(usable)  // 处理 Promise
  }
}
```

`useThenable` → `trackUsedThenable`，检查 Promise 状态：

```javascript
switch (thenable.status) {
  case 'fulfilled': return thenable.value   // 已完成，直接返回
  case 'rejected':  throw thenable.reason   // 已失败，抛出错误
  default:
    // pending：为 Promise 附加监听器，修改其 status
    thenable.then(
      value => { thenable.status = 'fulfilled'; thenable.value = value },
      err   => { thenable.status = 'rejected';  thenable.reason = err  },
    )
    throw SuspenseException   // 抛出特殊异常，中断渲染
}
```

注意：抛出的是 `SuspenseException`（一个固定对象），实际的 Promise 存在全局变量 `suspendedThenable` 里。

---

## 第二步：throwException 捕获异常

`performUnitOfWork` 抛出异常后，进入 `throwException`：

```javascript
function throwException(root, returnFiber, sourceFiber, value, renderLanes) {
  // 1. 标记抛出异常的 fiber
  sourceFiber.flags |= Incomplete

  // 2. 找到最近的 Suspense 边界（通过 Suspense 上下文栈）
  const suspenseBoundary = getSuspenseHandler()

  // 3. 标记边界需要捕获此次异常
  markSuspenseBoundaryShouldCapture(suspenseBoundary, ...)
  // → suspenseBoundary.flags |= ShouldCapture

  // 4. 把 Promise 加入边界的 RetryQueue
  if (suspenseBoundary.updateQueue === null) {
    suspenseBoundary.updateQueue = new Set([wakeable])
  } else {
    suspenseBoundary.updateQueue.add(wakeable)
  }

  // 5. 附加 ping 监听器（并发模式）
  attachPingListener(root, wakeable, renderLanes)
  // → wakeable.then(ping, ping)
}
```

`throwException` 之后，React **从 Suspense 边界重新渲染**（不是从根节点），这次 Suspense 发现自己有 `ShouldCapture` flag，切换到展示 fallback 的逻辑。

---

## 第三步：Suspense 展示 fallback

`updateSuspenseComponent` 里，Suspense 维护两套子树：

```
<Suspense fallback={<Spinner/>}>
  <Content/>
</Suspense>

对应的 Fiber 子树：
  Offscreen (mode: 'visible' 或 'hidden')
    └── <Content/>          ← 主内容
  <Spinner/>                 ← fallback（只在挂起时存在）
```

切换逻辑：

```javascript
const didSuspend = (workInProgress.flags & DidCapture) !== NoFlags

if (didSuspend) {
  showFallback = true
  workInProgress.flags &= ~DidCapture
}

if (showFallback) {
  // 主内容放入 Offscreen（mode: 'hidden'），不显示但保留 fiber
  // 渲染 fallback 子树
  workInProgress.memoizedState = SUSPENDED_MARKER
} else {
  // 正常渲染主内容
  workInProgress.memoizedState = null
}
```

主内容被放入 `Offscreen hidden` 而不是直接卸载，这样 Promise resolve 后可以直接复用已有 fiber，不用重建。

---

## 第四步：Promise resolve 后恢复

有两条恢复路径：

### 路径一：ping（渲染期间 resolve）

`throwException` 时通过 `attachPingListener` 绑定：

```javascript
wakeable.then(pingSuspendedRoot.bind(null, root, wakeable, lanes))
```

Promise resolve 时调用 `pingSuspendedRoot`：
- 标记 `root.pingedLanes |= lanes`
- 如果满足条件，直接中断当前渲染，从头重跑

### 路径二：retry（fallback 已提交后 resolve）

Commit Phase 提交 fallback 时，`attachSuspenseRetryListeners` 绑定：

```javascript
// 为 RetryQueue 里的每个 Promise 绑定 retry 回调
wakeable.then(resolveRetryWakeable.bind(null, boundaryFiber, wakeable))
```

Promise resolve 时调用 `resolveRetryWakeable`：
- 分配一个 **RetryLane**
- 调度新的渲染

### RetryLane

Suspense 重试专用的 Lane，共 4 个（RetryLane1~4，轮转使用）：

```javascript
let nextRetryLane = RetryLane1

function claimNextRetryLane() {
  const lane = nextRetryLane
  nextRetryLane <<= 1
  if ((nextRetryLane & RetryLanes) === 0) nextRetryLane = RetryLane1  // 循环
  return lane
}
```

优先级低于 DefaultLane，高于 IdleLane。

---

## 并发模式 vs 同步模式

| | 同步模式 | 并发模式 |
|--|--|--|
| fallback 展示 | 立即展示 | 可延迟 300ms（避免闪烁） |
| 渲染方式 | 同一次渲染中切换 | 独立的重试渲染 |
| ping 响应 | 不支持 | 可以中断当前渲染重来 |

并发模式下有 **FALLBACK_THROTTLE_MS（300ms）** 节流：如果 Promise 在 300ms 内 resolve，React 会等一等再展示 fallback，避免用户看到一闪而过的 loading。

---

## 完整流程

```
use(promise) / throw promise
      ↓
trackUsedThenable：promise pending → throw SuspenseException
      ↓
throwException：
  找到最近 Suspense 边界
  边界打 ShouldCapture flag
  把 promise 加入 RetryQueue
  attachPingListener（wakeable.then(ping)）
      ↓
从 Suspense 边界重新 beginWork
  发现 DidCapture → showFallback = true
  主内容 → Offscreen hidden（保留 fiber）
  渲染 fallback
      ↓
Commit：提交 fallback
  attachSuspenseRetryListeners（wakeable.then(retry)）
      ↓
Promise resolve
      ↓
resolveRetryWakeable
  分配 RetryLane
  调度新渲染
      ↓
重新渲染 Suspense：
  use(promise) → promise.status === 'fulfilled' → 返回值
  主内容正常渲染
  Offscreen hidden → visible
  移除 fallback
```
