# React 源码笔记

梳理 React 核心流程与实现原理。

## 目录

| 文件 | 内容 |
|------|------|
| [01-main-flow.md](./01-main-flow.md) | React 主流程：初次渲染、更新、Fiber 工作循环、Commit 阶段 |
| [02-hooks.md](./02-hooks.md) | Hooks 实现原理：Dispatcher、Hook 链表、useState、useEffect 等 |
| [03-diff.md](./03-diff.md) | Diff 算法：单节点、多节点两轮遍历、lastPlacedIndex 移动判断 |
| [04-render-phase.md](./04-render-phase.md) | Render Phase：双缓冲、beginWork/completeWork、bailout 优化 |
| [05-commit-phase.md](./05-commit-phase.md) | Commit Phase：三阶段、current 切换时机、useEffect 时序 |
| [06-scheduler.md](./06-scheduler.md) | Scheduler：双队列、小顶堆、时间片、MessageChannel、延迟任务 |
| [07-concurrent.md](./07-concurrent.md) | 异步可中断更新：时间片机制、高优先级插队、双缓冲保证正确性 |
| [08-events.md](./08-events.md) | 事件系统：委托到 root、合成事件、优先级映射、原生与 React 事件顺序 |
| [09-context.md](./09-context.md) | Context：_currentValue 栈管理、依赖链表、变化传播、为何绕过 memo |
| [10-lane.md](./10-lane.md) | Lane 模型：位掩码、各 Lane 含义、纠缠、饥饿处理、与 Scheduler 互转 |
| [11-suspense.md](./11-suspense.md) | Suspense：throw Promise 机制、fallback 切换、ping/retry 恢复、并发模式差异 |

## 关键包

| 包 | 职责 |
|----|------|
| `packages/react/` | 核心 API：`useState`、`memo` 等 |
| `packages/react-dom/` | DOM 渲染器：`createRoot` |
| `packages/react-reconciler/` | Fiber 协调引擎（核心） |
| `packages/scheduler/` | 调度器：优先级、时间分片 |
| `packages/react-dom-bindings/` | 事件系统 |
