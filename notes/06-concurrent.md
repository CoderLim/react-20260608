# 异步可中断更新

## 一句话

**时间片是手段，高优先级插队才是目的。**

---

## 为什么需要可中断

同步模式下，`setState` 触发后立刻同步渲染，主线程被占用期间用户交互无响应。

并发模式下，渲染交给 Scheduler 调度，每 5ms 让出一次主线程，让浏览器有机会处理用户事件。用户事件触发高优先级更新，才能插队打断低优先级渲染。

---

## 两层机制

### 异步：更新交给 Scheduler 排队

```
setState
  → scheduleUpdateOnFiber
  → ensureRootIsScheduled
  → scheduleCallback(priority, performConcurrentWorkOnRoot)
```

通过 MessageChannel 在下一个宏任务执行，不阻塞当前主线程。

### 可中断：时间片检查

```javascript
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYieldToHost()) {
    performUnitOfWork(workInProgress)
  }
}
```

每处理完一个 Fiber 节点检查一次，5ms 到了立刻退出。

---

## 中断后的两种结果

退出 workLoop 后，`renderRootConcurrent` 入口检查 lanes：

```javascript
if (workInProgressRoot !== root || workInProgressRootRenderLanes !== lanes) {
  prepareFreshStack(root, lanes)  // 重置，从根节点重新开始
}
```

| 情况 | workInProgress | 下次行为 |
|---|---|---|
| 没有高优先任务插队 | 保留断点 | 从断点继续，等于什么都没发生 |
| 有高优先任务插队 | 丢弃 | lanes 变了，prepareFreshStack，从根重跑 |

从断点继续只是"没有更重要的事"时的默认行为，高优先任务插队必然重跑。

---

## 时间片的真正作用

时间片暂停本质上是一次**主动检查窗口**：

```
渲染跑了 5ms → 让出主线程
  → 浏览器处理用户事件、绘制
  → 用户事件触发高优先更新（登记到调度队列）
  → Scheduler 再次调度 → 发现更高优先级
  → 插队，重新渲染
```

没有时间片：100ms 的渲染锁住主线程，用户点击无法被处理，高优先任务根本没机会被感知。

有了时间片：每 5ms 给浏览器一次处理事件的机会，高优先任务才能被登记、才能插队。

---

## 完整链路

```
低优先级渲染开始
  │
  ├─ 每 5ms 让出主线程
  │      ↓
  │   浏览器处理用户事件
  │      ↓
  │   高优先更新进入调度队列
  │      ↓
  │   ensureRootIsScheduled 发现更高优先级
  │      ↓
  │   取消当前 Scheduler 任务，重新调度
  │      ↓
  │   renderRootConcurrent：lanes 变了 → prepareFreshStack
  │      ↓
  │   从根节点重跑 Render Phase（低优先级已做的工作丢弃）
  │      ↓
  │   高优先级渲染完成 → Commit → 用户看到响应
  │      ↓
  └─ 重新调度低优先级渲染
```

---

## 双缓冲保证正确性

高优先级打断时，屏幕上展示的是 `current` 树对应的 DOM，始终稳定。

`workInProgress` 是独立构建的，丢了就丢了，不影响用户看到的内容。

```
current 树         → 屏幕上的 DOM（始终稳定）
workInProgress 树  → 正在计算的新树（可随时丢弃重建）
```

Render Phase 不操作 DOM，正是为了让"丢弃重来"没有副作用。
