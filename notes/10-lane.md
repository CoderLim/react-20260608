# Lane 模型

源码：`packages/react-reconciler/src/ReactFiberLane.js`

## 为什么用位掩码

Lane 是 32 位整数，每一位代表一种更新类型：

```javascript
export type Lane = number   // 单个 lane，只有一位为 1
export type Lanes = number  // 多个 lane 的集合，多位为 1
```

位运算的好处：
- 合并：`mergeLanes(a, b) = a | b`
- 包含检查：`(lanes & lane) !== 0`
- 排除：`lanes & ~lane`
- 取最高优先级：`lanes & -lanes`（取最低位，数值越小优先级越高）

---

## 各 Lane 的含义

```javascript
SyncLane              = 0b0000...0010   // 用户输入、最高优先级
InputContinuousLane   = 0b0000...1000   // 连续事件（mousemove、scroll）
DefaultLane           = 0b0010...0000   // 普通 setState
TransitionLane1-16    = 0b...           // startTransition 过渡更新（16 个）
RetryLane1-4          = 0b...           // Suspense 数据加载完成后重试
IdleLane              = 0b...           // 空闲时执行
OffscreenLane         = 0b...           // 离屏渲染（Activity/keep-alive）
```

数值越小，优先级越高。

---

## requestUpdateLane：setState 用哪个 Lane

```javascript
function requestUpdateLane(fiber) {
  // 非并发模式 → 始终同步
  if (!ConcurrentMode) return SyncLane

  // 渲染阶段触发的更新 → 复用当前渲染的 lane
  if (isRenderPhaseUpdate) return pickArbitraryLane(workInProgressRootRenderLanes)

  // startTransition 内 → 分配一个 TransitionLane
  const transition = requestCurrentTransition()
  if (transition !== null) return requestTransitionLane(transition)

  // 事件回调内 → 使用事件对应的优先级（click → SyncLane）
  if (currentUpdatePriority !== NoLane) return currentUpdatePriority

  // 其他（useEffect、setTimeout）→ DefaultLane
  return getCurrentEventPriority()
}
```

---

## getNextLanes：选出本次要渲染的 lanes

每次调度前，从 `root.pendingLanes` 里挑出最高优先级的 lanes：

```javascript
function getNextLanes(root, wipLanes) {
  const pendingLanes = root.pendingLanes
  if (pendingLanes === NoLanes) return NoLanes

  // 1. 去掉被 Suspense 挂起的 lanes
  const unblockedLanes = pendingLanes & ~suspendedLanes

  // 2. 取最高优先级
  nextLanes = getHighestPriorityLanes(unblockedLanes)

  // 3. 不随意打断当前正在渲染的 lanes
  if (wipLanes !== NoLanes && nextLane >= wipLane) {
    return wipLanes  // 新任务优先级不高于当前，继续当前渲染
  }

  // 4. 加入纠缠的 lanes（见下文）
  nextLanes |= entanglements[laneIndex]

  return nextLanes
}
```

---

## Lane 纠缠（Entanglement）

**含义**：纠缠的 lanes 必须一起渲染，不能只提交其中一部分。

### 为什么需要多个 TransitionLane

如果只有一个 TransitionLane，所有 transition 共享它，就会被视为同一批，互相阻塞。

```javascript
function SearchPage() {
  const [query, setQuery] = useState('')   // 搜索词，渲染慢
  const [theme, setTheme] = useState('')   // 主题，渲染快
}

startTransition(() => setQuery('hello'))  // 如果只有一个 lane
startTransition(() => setTheme('dark'))   // 必须等搜索渲染完才能提交主题
```

多个 TransitionLane 让**不同 state 的 transition 能并行推进**：

```
setQuery('hello') → TransitionLane1   独立调度，慢慢跑
setTheme('dark')  → TransitionLane2   独立调度，快速提交
```

### 纠缠只发生在同一个队列

```javascript
// 同一 state 快速输入 → 同一队列 → 互相纠缠，必须一起提交
startTransition(() => setQuery('a'))    // TransitionLane1 ┐
startTransition(() => setQuery('ab'))   // TransitionLane2 ├ 纠缠
startTransition(() => setQuery('abc'))  // TransitionLane3 ┘

// 不同 state → 不同队列 → 不纠缠，独立运行
startTransition(() => setTheme('dark')) // TransitionLane4，不受上面影响
```

纠缠防止同一 state 的旧结果覆盖新结果（用户看到搜索词倒退）。

```javascript
function entangleLanes(root, a, b) {
  root.entangledLanes |= a | b
  entanglements[indexOfA] |= b   // a 的纠缠集包含 b
  entanglements[indexOfB] |= a   // b 的纠缠集包含 a
}
```

### 16 个 TransitionLane 用完了怎么办

TransitionLane 是循环分配的，用完 16 个后回到 Lane1：

```javascript
let nextTransitionLane = TransitionLane1

function claimNextTransitionLane() {
  const lane = nextTransitionLane
  nextTransitionLane <<= 1
  if ((nextTransitionLane & TransitionLanes) === 0) {
    nextTransitionLane = TransitionLane1  // 循环回到第一个
  }
  return lane
}
```

复用已有 Lane 意味着新 transition 和原 Lane 上未完成的任务**退化为同一批**，两者被强制一起渲染。这是一种降级策略——略微损失独立性，但不影响正确性。

实践中触发这种情况需要同时有 16 个以上未完成的 transition，极难出现。

### 其他触发纠缠的场景

纠缠不只发生在 transition 上：

| 场景 | 纠缠的 lanes | 原因 |
|---|---|---|
| 同一 state 的多次 transition | TransitionLane 之间 | 防止旧状态覆盖新状态 |
| InputContinuousLane + DefaultLane 同时存在 | 自动合并到一起渲染 | 保证 UI 一致性（拖动 + 数据更新同帧提交） |
| 同一 Suspense 边界多个 Promise resolve | RetryLane 之间 | 数据全部就绪再一起切换，避免部分显示 |

---

## 饥饿处理（Starvation）

低优先级任务可能被高优先级任务不断打断，永远无法执行。React 用过期机制解决：

```javascript
function markStarvedLanesAsExpired(root, currentTime) {
  let lanes = root.pendingLanes
  while (lanes > 0) {
    const lane = 1 << pickArbitraryLaneIndex(lanes)
    const expirationTime = expirationTimes[index]

    if (expirationTime === NoTimestamp) {
      // 首次发现等待，记录过期时间
      expirationTimes[index] = computeExpirationTime(lane, currentTime)
    } else if (expirationTime <= currentTime) {
      // 已超时，加入 expiredLanes
      root.expiredLanes |= lane
    }
    lanes &= ~lane
  }
}
```

过期时间（从首次等待开始计算）：
| Lane | 过期时间 |
|---|---|
| SyncLane / InputContinuousLane | 250ms |
| DefaultLane / TransitionLane | 5000ms |
| IdleLane / OffscreenLane | 永不过期 |

进入 `expiredLanes` 后，下次执行时**切换为同步模式**，不受时间片约束，强制跑完。

---

## Lane 和 Scheduler 优先级的互转

两套优先级系统需要互转：

| Lane | Scheduler Priority |
|---|---|
| SyncLane | ImmediatePriority（立即执行） |
| InputContinuousLane | UserBlockingPriority（250ms） |
| DefaultLane / TransitionLane | NormalPriority（5s） |
| IdleLane | IdlePriority（永不超时） |

---

## FiberRoot 上的 Lane 字段

```javascript
root.pendingLanes      // 所有待渲染的 lanes
root.suspendedLanes    // 被 Suspense 挂起的 lanes
root.pingedLanes       // 准备好重试的 lanes（数据加载完成）
root.expiredLanes      // 已过期、需立即同步执行的 lanes
root.expirationTimes   // 每个 lane 的过期时间（数组，按 lane index）
root.entangledLanes    // 有纠缠关系的 lanes
root.entanglements     // 每个 lane 的纠缠集合（数组，按 lane index）
```

---

## Lane 的生命周期

```
setState
  → requestUpdateLane  分配 lane
  → markRootUpdated    root.pendingLanes |= lane
  → ensureRootIsScheduled
      → getNextLanes   选最高优先级
      → lanesToSchedulerPriority  转换为 Scheduler 优先级
      → scheduleCallback(priority, performConcurrentWorkOnRoot)

每次渲染后
  → markStarvedLanesAsExpired  检查是否有 lane 即将饥饿

Commit 结束
  → markRootFinished  root.pendingLanes &= ~finishedLanes  清除已完成
  → ensureRootIsScheduled  若还有 pendingLanes，继续调度
```
