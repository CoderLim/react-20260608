# Scheduler

源码：`packages/scheduler/src/forks/Scheduler.js`

## 整体设计：双队列

```javascript
var taskQueue  = []  // 立即执行的任务，按 expirationTime 排序
var timerQueue = []  // 延迟执行的任务，按 startTime 排序
```

两个队列都是**小顶堆**（MinHeap），堆顶始终是最高优先级的任务：
- `peek`：O(1) 取堆顶
- `push / pop`：O(log n)

分成两个队列：延迟任务到期前不需要参与调度，单独管理，到期后搬到 taskQueue。

---

## Task 结构

```javascript
{
  id,              // 递增 id，sortIndex 相同时按插入顺序
  callback,        // 要执行的函数（返回 function 表示未完成）
  priorityLevel,   // 优先级
  startTime,       // 何时可以开始执行
  expirationTime,  // 超时时间 = startTime + timeout
  sortIndex,       // 堆排序 key：taskQueue 用 expirationTime，timerQueue 用 startTime
}
```

---

## 优先级与超时时间

| 优先级 | timeout | 用途 |
|---|---|---|
| Immediate | -1ms | 创建即过期，无条件执行 |
| UserBlocking | 250ms | 用户交互 |
| Normal | 5000ms | 普通更新 |
| Low | 10000ms | 低优先级 |
| Idle | 永不超时 | 空闲时执行 |

`expirationTime = startTime + timeout`

Immediate 的 timeout 是 -1，创建时就已过期，不受时间片限制。

---

## scheduleCallback：任务入队

```
startTime > currentTime  → 延迟任务，进 timerQueue，设置 setTimeout 等待到期
startTime <= currentTime → 立即任务，进 taskQueue，触发 requestHostCallback
```

`requestHostCallback` 通过 **MessageChannel** 异步触发工作循环。

---

## 工作循环 workLoop

```javascript
function workLoop(currentTime) {
  advanceTimers(currentTime)        // 把 timerQueue 里到期的任务搬到 taskQueue

  currentTask = peek(taskQueue)
  while (currentTask !== null) {

    if (currentTask.expirationTime > currentTime && shouldYieldToHost()) {
      break                         // 任务未过期 且 时间片用完 → 让出主线程
    }

    const continuation = currentTask.callback(didTimeout)

    if (typeof continuation === 'function') {
      currentTask.callback = continuation   // 任务未完，保留，下次继续
      return true                           // 还有工作
    } else {
      pop(taskQueue)                        // 任务完成，出堆
    }

    currentTask = peek(taskQueue)
  }

  return currentTask !== null              // 是否还有剩余任务
}
```

两个关键点：
1. **已过期的任务不受时间片限制**（`expirationTime <= currentTime`），会强制执行到完成
2. **callback 可以返回函数**，表示"还没做完，下次接着"——这是 React Render Phase 可中断的底层支撑

---

## 时间片：shouldYieldToHost

```javascript
function shouldYieldToHost() {
  const timeElapsed = getCurrentTime() - startTime
  return timeElapsed >= frameInterval   // frameInterval 默认 5ms
}
```

每 5ms 检查一次，超过就让出主线程。

---

## 为什么用 MessageChannel 而不是 setTimeout

`setTimeout(fn, 0)` 有浏览器强制的 **4ms 最小延迟**，Scheduler 每次时间片用完都要重新调度，4ms 的开销累积起来不可忽视。

MessageChannel 的 `port.postMessage` 没有这个限制，当前 JS 执行栈清空后**立即**进入下一个宏任务，延迟接近 0。

```javascript
const channel = new MessageChannel()
channel.port1.onmessage = performWorkUntilDeadline
schedulePerformWorkUntilDeadline = () => port2.postMessage(null)
```

降级策略：`setImmediate`（Node.js）> `MessageChannel` > `setTimeout`

---

## 延迟任务搬运：advanceTimers

每次 workLoop 开始前执行，把 timerQueue 里 `startTime <= currentTime` 的任务搬到 taskQueue：

```javascript
function advanceTimers(currentTime) {
  let timer = peek(timerQueue)
  while (timer !== null && timer.startTime <= currentTime) {
    pop(timerQueue)
    timer.sortIndex = timer.expirationTime   // 换排序 key！
    push(taskQueue, timer)
    timer = peek(timerQueue)
  }
}
```

sortIndex 从 `startTime` 换成 `expirationTime`，因为两个队列的排序依据不同。

---

## Scheduler 的 task 在 React 里是什么

Scheduler 是通用调度器，不感知 React 内部细节。React 调度时传入的 callback 是：

```javascript
scheduleCallback(priority, performConcurrentWorkOnRoot.bind(null, root))
```

所以 **Scheduler 里的一个 task = 对某个 root 跑一轮工作循环**。不是一个组件，不是一个 Fiber 节点。

**多次 setState 只对应一个 task**

```javascript
setState(1)
setState(2)
setState(3)
```

每次 setState 都调用 `ensureRootIsScheduled`，但调度器发现已有针对该 root 的 task，不重复创建，三次更新合并到同一个 task 处理。

**一个 task 可能执行多次**

时间片用完，workLoop 退出，`performConcurrentWorkOnRoot` 返回自身作为 continuation：

```javascript
if (root.callbackNode === originalCallbackNode) {
  return performConcurrentWorkOnRoot.bind(null, root)  // 还没跑完，下次继续
}
```

Scheduler 看到返回值是函数，保留该 task，下个时间片继续执行。一次逻辑渲染可能跨多个时间片、对应同一个 task 被执行多次。

---

## 完整时序

```
scheduleCallback(priority, callback)
  │
  ├─ 立即任务 → push(taskQueue)
  │               → requestHostCallback
  │                       → port.postMessage
  │                               → performWorkUntilDeadline
  │                                       → workLoop()
  │                                   ┌──────────────────────┐
  │                                   │ advanceTimers        │
  │                                   │ peek(taskQueue)      │
  │                                   │ shouldYieldToHost?   │
  │                                   │   是 → break         │
  │                                   │   否 → 执行 callback  │
  │                                   │     返回 fn → 下次继续 │
  │                                   │     返回空 → pop 出堆  │
  │                                   └──────────────────────┘
  │                                   有剩余 → 再 postMessage
  │                                   无剩余 → 检查 timerQueue
  │
  └─ 延迟任务 → push(timerQueue)
                  → requestHostTimeout(delay)
                          → handleTimeout
                                  → advanceTimers
                                  → push(taskQueue)
                                  → requestHostCallback
```
