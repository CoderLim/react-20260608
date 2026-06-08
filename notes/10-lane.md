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

**典型场景**：对同一状态队列的多个 transition 更新。

```javascript
// 快速连续触发 startTransition
startTransition(() => setQuery('a'))   // → TransitionLane1
startTransition(() => setQuery('ab'))  // → TransitionLane2，与 Lane1 纠缠
startTransition(() => setQuery('abc')) // → TransitionLane3，与 Lane1/2 纠缠
```

三个 lane 互相纠缠后，渲染任何一个时，其他两个也会被加入本次渲染。这样用户永远只会看到最新的输入结果，不会看到中间状态。

```javascript
function entangleLanes(root, a, b) {
  root.entangledLanes |= a | b
  // a 的纠缠集包含 b，b 的纠缠集包含 a
  entanglements[indexOfA] |= b
  entanglements[indexOfB] |= a
}
```

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
