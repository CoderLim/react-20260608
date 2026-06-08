# Render Phase

源码：`packages/react-reconciler/src/ReactFiberWorkLoop.js`

## 目标

纯计算，**不操作 DOM**。算出哪些节点需要变更，产出带 flags 的 workInProgress Fiber 树。

---

## 双缓冲 Fiber 树

React 始终维护两棵树：

```
current 树         ← 当前屏幕展示的
workInProgress 树  ← 正在构建的新树
```

每个 Fiber 节点通过 `alternate` 互指对方。Render Phase 在 workInProgress 上工作，Commit Phase 结束后整体切换（`root.current = finishedWork`）。

好处：workInProgress 随时可以丢弃，不影响屏幕。高优先级任务插队时直接重建 workInProgress，无副作用。

---

## 工作循环

```javascript
// 同步：一口气跑完，不可中断
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress)
  }
}

// 并发：每个节点后检查时间片，可中断
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYieldToHost()) {
    performUnitOfWork(workInProgress)
  }
}
```

---

## 深度优先遍历：beginWork ↓ + completeWork ↑

```
     App
    /   \
  Header  Main
  /
Nav

beginWork(App)
  beginWork(Header)
    beginWork(Nav)
    completeWork(Nav)      ← 叶子节点，立即 complete
  completeWork(Header)
  beginWork(Main)
  completeWork(Main)
completeWork(App)
```

先一路向下 beginWork，到叶子节点后 completeWork 向上回溯，遇到兄弟节点再向下。

---

## beginWork

根据 Fiber tag 分发，执行组件逻辑产出子节点：

| Fiber tag | 处理方式 |
|---|---|
| 函数组件 | 调用函数，执行 Hooks |
| 类组件 | 调用 `render()` |
| HostComponent（div 等） | 处理 props，reconcile children |
| Context.Provider | 更新 context 值 |

执行后调用 `reconcileChildren` 进行 Diff，给变更节点打 flags：

```
Placement   需要插入
Update      需要更新
Deletion    需要删除
```

### mountChildFibers vs reconcileChildFibers

`reconcileChildren` 根据是否有 `current` 选择不同实现：

```javascript
current === null
  ? mountChildFibers(...)       // 首次渲染
  : reconcileChildFibers(...)   // 更新渲染
```

区别：**`mountChildFibers` 不追踪 Placement effect**。

首次渲染时子树全是新建的，如果每个节点都打 Placement flag，Commit Phase 就要逐个 `appendChild`，性能差。实际做法是只给根节点打一次 Placement，一次性把整棵 DOM 子树插入，子节点本身不打 flag。

---

## bailout：跳过不必要的渲染

beginWork 里，如果判断当前 Fiber 不需要更新，直接复用旧子树，跳过整个子树的渲染：

```javascript
// 满足以下所有条件才能 bailout：
// 1. props 没有变化（新旧 props 引用相同）
// 2. context 没有变化
// 3. 该 Fiber 上没有待处理的更新
if (hasNoChange) {
  return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes)
  // → 直接复用 current 的子 Fiber，不重新执行组件函数
}
```

触发 bailout 的常见场景：
- `React.memo`：对比前后 props，相同则 bailout
- `shouldComponentUpdate` 返回 false
- `PureComponent` 浅比较 props/state 无变化
- `useState` 的 setState 传入相同值（eagerState 优化）

bailout 是 React 性能优化的核心机制，决定了"哪些组件不用重渲染"。

---

## completeWork

向上回溯时执行：

**首次渲染**
- 创建真实 DOM 实例（`document.createElement`）
- 设置初始属性、绑定事件
- 把子节点的 DOM 拼装成 DOM 子树（`appendAllChildren`）

**更新渲染**
- 对比新旧 props，把差异收集到 `fiber.updateQueue`（数组形式：`[key, value, key, value...]`）
- 有差异则给 fiber 打 `Update` flag

**两种情况都会执行：**`bubbleProperties`，把子树的 flags 和 lanes 向上合并到 `subtreeFlags`，供 Commit Phase 快速定位需要操作的节点。

---

## Render Phase 的产出

Render Phase 结束时，`root.finishedWork` 是新 workInProgress 树的根节点，带有：

- 每个节点的 `flags`：该节点自身需要做什么操作
- 每个节点的 `subtreeFlags`：子树中是否有需要操作的节点
- 函数组件的 `fiber.updateQueue.lastEffect`：effect 循环链表
- HostComponent 的 `fiber.updateQueue`：需要更新的 props 差异

交给 Commit Phase 消费。
