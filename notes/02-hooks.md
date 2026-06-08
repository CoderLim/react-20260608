# Hooks 实现原理

## 核心：两个设计

### 1. Dispatcher —— 同一个 API，不同阶段不同实现

`useState` 等 API 本身没有逻辑，只是转发给当前 Dispatcher：

```
首次渲染  → HooksDispatcherOnMount  → mountState
更新渲染  → HooksDispatcherOnUpdate → updateState
渲染中更新 → HooksDispatcherOnRerender
```

切换时机：`renderWithHooks` 开始时，根据 `current.memoizedState` 是否存在来判断。

### 2. Hook 链表 —— 用顺序绑定状态

组件所有 Hook 按调用顺序形成单向链表，挂在 `fiber.memoizedState`：

```
fiber.memoizedState → Hook#0 → Hook#1 → Hook#2 → null
                     useState  useEffect  useMemo
```

- **首次渲染**：依次创建节点，追加到链表
- **更新渲染**：依次取出旧节点，复用或计算新值

**这就是为什么不能在条件语句中调用 Hook**：顺序一变，新旧链表错位，状态对应关系就乱了。

---

## 各 Hook 实现要点

### useState / useReducer

- 状态存在 `hook.memoizedState`
- `setState` 把 update 对象加入 `hook.queue.pending`（循环链表）
- 更新时遍历 pending 队列，按优先级计算最终 state
- **优化**：setState 时提前算出新 state，如果与旧值相同（`Object.is`），直接跳过调度

### useEffect / useLayoutEffect

- Effect 对象存在 `hook.memoizedState`，同时加入 `fiber.updateQueue` 的循环链表
- 更新时对比 deps（`Object.is` 逐个比较）：
  - deps 不变 → 不设 `HookHasEffect` 标记，Commit 阶段跳过执行
  - deps 改变 → 设 `HookHasEffect`，Commit 阶段执行

| | useEffect | useLayoutEffect |
|---|---|---|
| 执行时机 | 浏览器绘制后（异步） | DOM 更新后、绘制前（同步） |

### useRef

- 只是把 `{ current: initialValue }` 存入 `memoizedState`
- 每次更新返回同一个对象引用，不触发重渲染

### useMemo / useCallback

- `memoizedState = [value, deps]`
- 更新时对比 deps，未变则直接返回缓存值
- `useCallback(fn, deps)` 就是 `useMemo(() => fn, deps)`

### useContext

- 不创建 Hook 节点，直接读 `context._currentValue`
- 同时在 `fiber.dependencies` 上记录依赖，Provider 值变化时遍历订阅者触发更新

---

## 各 Hook 的 memoizedState

| Hook | memoizedState |
|------|---------------|
| useState / useReducer | state 值 |
| useEffect / useLayoutEffect | Effect 对象 |
| useRef | `{ current }` 对象 |
| useMemo / useCallback | `[value, deps]` |
| useContext | 无（不占链表节点） |
