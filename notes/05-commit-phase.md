# Commit Phase

源码：`packages/react-reconciler/src/ReactFiberCommitWork.js`

## 特点

**同步执行，不可中断**。拿到 Render Phase 产出的 `finishedWork`，操作真实 DOM，执行副作用。

---

## 如何遍历 effects

Commit Phase 不重新遍历整棵 Fiber 树，而是利用 Render Phase 收集的 `subtreeFlags` 快速定位：

```javascript
// 某个子树根节点
if ((subtreeFlags & MutationMask) === NoFlags) {
  // 子树里没有需要 Mutation 的节点，直接跳过整棵子树
  return
}
```

只有 `subtreeFlags` 不为 0 的子树才会往下走，大幅减少遍历量。

---

## 三个子阶段

### BeforeMutation（DOM 变更前）

- 执行 `getSnapshotBeforeUpdate`（类组件，读取变更前的 DOM 快照）
- 清理旧的 ref（`ref.current = null`）

### Mutation（操作 DOM）

```
Placement → appendChild / insertBefore
Update    → 更新 DOM 属性（消费 fiber.updateQueue 里的 props 差异）
Deletion  → removeChild，同时执行 componentWillUnmount / useEffect cleanup
```

- 执行 `useInsertionEffect`（在 DOM 变更前插入样式，CSS-in-JS 方案使用）
- **Mutation 结束后，`root.current` 切换为 `finishedWork`**

### Layout（DOM 已更新，绘制前）

- 执行 `componentDidMount` / `componentDidUpdate`
- 执行 `useLayoutEffect` 的 destroy，再执行 create（同步，阻塞绘制）
- 更新 ref（`ref.current = DOM 节点`）

---

## current 树的切换时机

`root.current = finishedWork` 发生在 **Mutation 结束、Layout 开始之前**。

这个时机是刻意的：

- **Mutation 阶段**（切换前）：`current` 还是旧树 → `componentWillUnmount` 读到的是旧 state
- **Layout 阶段**（切换后）：`current` 已是新树 → `componentDidMount` 读到的是新 state

如果在 Layout 之后切换，`componentDidMount` 里调用 `setState` 时 React 会找不到正确的 fiber。

---

## useEffect 的执行时序

Commit Phase 本身**不执行** useEffect，只在结束时登记：

```javascript
// Commit 结束时
scheduleCallback(NormalPriority, flushPassiveEffects)
```

实际执行在浏览器绘制完成后，由 Scheduler 异步调度：

```
Commit Phase（同步）
  BeforeMutation → Mutation → root.current 切换 → Layout
  scheduleCallback(flushPassiveEffects)    ← 仅登记

浏览器绘制

flushPassiveEffects（异步）
  所有组件 useEffect destroy（批量）
  所有组件 useEffect create（批量）
```

destroy 和 create **不配对**，先把所有组件的 destroy 全跑完，再跑所有 create：

```
A.destroy → B.destroy → C.destroy → A.create → B.create → C.create
```

---

## 例外：强制同步执行 flushPassiveEffects

下次 Commit 开始时，若上一轮 `flushPassiveEffects` 还没执行，会在 `commitRoot` 入口强制同步执行：

```javascript
if (rootWithPendingPassiveEffects !== null) {
  flushPassiveEffects()  // 先清掉上次的 useEffect
}
```

目的：保证新渲染开始前，上一轮的 useEffect 已经执行完。

---

## flushPassiveEffects 内部的 setState

`flushPassiveEffects` 同步执行完所有 create，但 create 里的 setState **不会中途触发新渲染**：

```
flushPassiveEffects()
  A.create  ← 里面 setState，登记更新，但不立即执行
  B.create
  C.create
返回
  ↓
新的 Render + Commit
  ↓
再次 scheduleCallback(flushPassiveEffects)
```

---

## useLayoutEffect vs useEffect

| | useLayoutEffect | useEffect |
|--|--|--|
| 执行时机 | DOM 更新后、绘制前 | 浏览器绘制后 |
| 是否阻塞绘制 | 是（同步） | 否（异步） |
| destroy/create 顺序 | 配对（先 destroy 再 create） | 批量 destroy，再批量 create |
| 适合场景 | 读写 DOM、避免闪烁 | 数据请求、事件订阅 |

---

## 整体时序

```
commitRoot(root)
  │
  ├─ 若有未执行的 passive effects → 强制 flushPassiveEffects
  │
  ├─ BeforeMutation
  │   └─ getSnapshotBeforeUpdate、清理 ref
  │
  ├─ Mutation
  │   └─ 操作 DOM（增/改/删）、useInsertionEffect
  │
  ├─ root.current = finishedWork   ← current 树切换
  │
  ├─ Layout
  │   └─ componentDidMount/Update、useLayoutEffect、更新 ref
  │
  └─ scheduleCallback(flushPassiveEffects)
          ↓（浏览器绘制后异步执行）
     flushPassiveEffects
       └─ 所有 useEffect destroy → 所有 useEffect create
```
