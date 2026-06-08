# React 主流程

## 整体脉络

```
createRoot().render()
      │
      ▼
  scheduleUpdateOnFiber      ← setState 也走这里
      │
      ▼
  Scheduler 调度
      │
      ▼
  Render Phase               构建 Fiber 树（不动 DOM）
      │
      ▼
  Commit Phase               操作真实 DOM + 执行副作用
```

---

## Render Phase：构建 Fiber 树

深度优先遍历，每个节点经过两步：

```
performUnitOfWork(fiber)
  ├─ beginWork   ↓ 向下：执行组件函数，Diff 子节点，返回子 Fiber
  └─ completeWork ↑ 向上：创建 DOM 实例，收集 effects
```

- **同步模式**：一口气跑完
- **并发模式**：可在 `workLoopConcurrent` 中被中断，让浏览器处理高优先级任务

---

## Commit Phase：三个子阶段

| 阶段 | 做什么 |
|------|--------|
| **BeforeMutation** | `getSnapshotBeforeUpdate`，读取 DOM 快照 |
| **Mutation** | 真正操作 DOM：增 / 改 / 删 |
| **Layout** | `componentDidMount/Update`、`useLayoutEffect`（同步） |

> `useEffect` 不在 Commit 内执行，而是在浏览器绘制完成后由 Scheduler 异步调度。

---

## 优先级：Lane 模型

用位掩码表示更新优先级，高优先级可以打断低优先级的渲染。

```
SyncLane       用户输入（最高）
DefaultLane    普通事件
TransitionLane useTransition 过渡更新
IdleLane       空闲时执行（最低）
```

---

## 关键文件

| 文件 | 职责 |
|------|------|
| `ReactFiberWorkLoop.js` | 工作循环主入口、commitRoot |
| `ReactFiberBeginWork.js` | beginWork：处理各类组件 |
| `ReactFiberCompleteWork.js` | completeWork：创建 DOM |
| `ReactFiberCommitWork.js` | Commit 三阶段实现 |
| `ReactChildFiber.js` | Diff 算法 |
| `ReactFiberLane.js` | 优先级管理 |
