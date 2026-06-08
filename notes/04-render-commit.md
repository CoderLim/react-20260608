# Render Phase 与 Commit Phase

## Render Phase

### 目标

纯计算，**不操作 DOM**。算出哪些节点需要变更，产出带 flags 的 workInProgress Fiber 树。

### 双缓冲 Fiber 树

React 始终维护两棵树：

```
current 树         ← 当前屏幕展示的
workInProgress 树  ← 正在构建的新树
```

每个 Fiber 节点通过 `alternate` 指向对方。Render Phase 在 workInProgress 上工作，完成后整体切换。

好处：workInProgress 随时可以丢弃，不影响屏幕。并发模式中断后，直接重建 workInProgress 即可。

### 工作循环

```javascript
// 同步：一口气跑完
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress)
  }
}

// 并发：时间片用完就让出主线程
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress)
  }
}
```

`shouldYield()` 来自 Scheduler，默认时间片 5ms。

### 深度优先遍历

每个节点经过 `performUnitOfWork`，分两步：

```
beginWork   ↓ 向下：执行组件逻辑，Diff 子节点，给变更节点打 flags
completeWork ↑ 向上：首次渲染创建 DOM 实例；更新时收集 props 变更
```

执行顺序示例：

```
     App
    /   \
  Header  Main
  /
Nav

beginWork(App) → beginWork(Header) → beginWork(Nav)
→ completeWork(Nav) → completeWork(Header) → beginWork(Main)
→ completeWork(Main) → completeWork(App)
```

先一路向下 beginWork，到叶子节点后 completeWork 向上回溯，遇到兄弟节点再向下。

### beginWork

根据 Fiber tag 分发，核心是执行组件逻辑产出子节点：

| Fiber tag | 处理方式 |
|---|---|
| 函数组件 | 调用函数，执行 Hooks |
| 类组件 | 调用 `render()` |
| HostComponent（div 等） | 处理 props，diff children |
| Context.Provider | 更新 context 值 |

执行后调用 `reconcileChildren`（Diff 算法），对比新旧子节点，给需要变更的节点打 flags：

```
Placement   需要插入
Update      需要更新
Deletion    需要删除
```

### completeWork

向上回溯时执行：
- **首次渲染**：创建真实 DOM 实例，设置属性、绑定事件
- **更新渲染**：对比新旧 props，把需要更新的属性收集到 `updateQueue`

然后调用 `bubbleProperties`：把子树的 flags 和 lanes **向上冒泡**到父节点的 `subtreeFlags`。

---

## Effects：副作用的收集与执行

"effects" 有两种，含义不同：

### 1. subtreeFlags 冒泡

completeWork 回溯时，每个节点把子树的 flags 合并到自身的 `subtreeFlags`：

```
completeWork(Nav)    → Nav.subtreeFlags 包含 Nav 子树的 flags
completeWork(Header) → Header.subtreeFlags |= Nav.subtreeFlags
completeWork(App)    → App.subtreeFlags |= Header.subtreeFlags + Main.subtreeFlags
```

Commit Phase 遍历时，如果某节点 `subtreeFlags === 0`，整个子树直接跳过，不用往下走。

### 2. effect 循环链表

有副作用 Hook 的函数组件，`fiber.updateQueue.lastEffect` 挂着一条循环链表：

```
fiber.updateQueue.lastEffect
        │
        ▼
  Effect(useLayoutEffect) → Effect(useEffect) → ... → （回到头，循环）
```

每个 Effect 对象带有 `tag` 标记，`HookHasEffect` 表示本次需要执行。

---

## Commit Phase

Commit Phase **同步执行，不可中断**。

### 三个子阶段

**BeforeMutation**
- 执行 `getSnapshotBeforeUpdate`（读 DOM 快照）

**Mutation**
- 真正操作 DOM：`appendChild` / `updateProperties` / `removeChild`
- 执行 `useInsertionEffect`

**Layout**
- DOM 已更新，浏览器还未绘制
- 执行 `componentDidMount` / `componentDidUpdate`
- 执行 `useLayoutEffect`（同步，会阻塞绘制）
- 在 Commit 结束时：`scheduleCallback(flushPassiveEffects)` 登记 useEffect，但不执行

### 时序总览

```
Commit Phase（同步）
  BeforeMutation
  Mutation              ← DOM 操作
  Layout                ← useLayoutEffect 同步执行完
  scheduleCallback(flushPassiveEffects)   ← 仅登记，不执行
Commit 结束

浏览器绘制

flushPassiveEffects（异步）
  所有组件的 useEffect destroy（批量）
  所有组件的 useEffect create（批量）
```

注意：destroy 和 create **不是配对执行**，而是先把所有组件的 destroy 全跑完，再跑所有 create：

```
顺序：A.destroy → B.destroy → C.destroy → A.create → B.create → C.create
```

### useEffect 的例外情况

若下次 Commit 开始时上一轮 `flushPassiveEffects` 还没执行，会在新 Commit 入口处**强制同步执行**：

```javascript
// commitRoot 开头
if (rootWithPendingPassiveEffects !== null) {
  flushPassiveEffects()  // 先清掉上次的 useEffect
}
```

### flushPassiveEffects 的执行

`flushPassiveEffects` 被调用时同步跑完所有 create，但 **create 内部的 setState 不会立即触发新渲染**：

```
flushPassiveEffects()
  A.destroy, B.destroy, C.destroy
  A.create   ← 里面 setState，登记一次更新
  B.create
  C.create
flushPassiveEffects 返回
  ↓
新的 Render + Commit（处理 setState 的更新）
  ↓
再次 scheduleCallback(flushPassiveEffects)
```

setState 调度的新渲染在 `flushPassiveEffects` 结束后才发生，不会中途插队。

---

## useLayoutEffect vs useEffect

| | useLayoutEffect | useEffect |
|--|--|--|
| 执行时机 | DOM 更新后、绘制前 | 浏览器绘制后 |
| 是否阻塞绘制 | 是（同步） | 否（异步） |
| destroy/create | 配对执行 | 批量 destroy，再批量 create |
| 适合场景 | 读写 DOM、避免闪烁 | 数据请求、事件订阅 |
