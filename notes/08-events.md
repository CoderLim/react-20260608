# 事件系统

源码：`packages/react-dom-bindings/src/events/`

## 事件委托：绑在 root 上

React 不把事件监听器绑在每个 DOM 节点上，而是统一绑在 **root container**（`createRoot` 传入的节点）：

```javascript
// createRoot 时调用
listenToAllSupportedEvents(rootContainerElement)
```

遍历所有支持的原生事件，每个事件在 root 上注册 **capture 和 bubble 两个监听器**。

好处：无论有多少组件，监听器数量固定，不随组件树增长。

---

## 两类事件：委托 vs 非委托

**大多数事件**：委托到 root

**非委托事件**（不冒泡，无法委托）：直接绑在对应 DOM 节点上
- `scroll`、`load`、`toggle`、`cancel`、`close`
- 媒体事件：`play`、`pause`、`ended`、`error` 等（共 32 个）

---

## 事件触发的完整链路

```
用户点击 DOM 节点
      ↓
root 的监听器触发
      ↓
根据事件类型确定优先级，包装成对应 dispatcher
      ↓
从触发的 DOM 节点找到对应 Fiber（通过 __reactFiber$ 属性缓存）
      ↓
从该 Fiber 向上遍历 Fiber 树，收集路径上所有匹配的监听器
      ↓
processDispatchQueue：按顺序执行回调
  capture 阶段：根 → 目标（倒序遍历）
  bubble 阶段：目标 → 根（正序遍历）
      ↓
回调在 batchedUpdates 内执行（批量处理 setState）
```

---

## 监听器收集：从 Fiber 树上摘

React 事件不是从 DOM 树找 `addEventListener`，而是从 **Fiber 树**上找 `onClick` prop：

```javascript
// 从目标 Fiber 一路向上
while (instance !== null) {
  const listener = getListener(instance, 'onClick')  // 从 props 取
  if (listener) listeners.push(...)
  instance = instance.return  // 向上遍历
}
```

这就是为什么 React 能实现自己的"冒泡"——和 DOM 结构无关，走的是 Fiber 树。

---

## 合成事件（SyntheticEvent）

对原生事件的封装，统一不同浏览器的差异：

```javascript
{
  nativeEvent,      // 原生事件对象
  target,           // 触发节点
  currentTarget,    // 当前处理节点（执行期间动态更新）
  type,
  stopPropagation() { ... },
  preventDefault() { ... },
  // 规范化属性（如 keyCode → key）
}
```

**现代 React 已移除对象池**，`persist()` 是空方法，事件对象可以安全地在异步中使用。

### 为什么移除对象池

早期对象池是为了减少 GC，但：
- 现代 JS 引擎对短命小对象的 GC 已很高效，对象池收益微乎其微
- 带来了隐藏陷阱：回调执行完对象被清空，异步访问 `e.target` 会得到 null
- 开发者必须手动调 `e.persist()`，API 不直觉
- React 18 自动批处理后异步场景更多，问题更突出

---

## 事件优先级

不同事件映射到不同 Lane，决定渲染时机：

| 优先级 | 典型事件 | Lane |
|---|---|---|
| DiscreteEventPriority | click、keydown、input、focus | SyncLane（最高） |
| ContinuousEventPriority | mousemove、scroll、wheel、drag | InputContinuousLane |
| DefaultEventPriority | 其他 | DefaultLane |

点击等交互事件优先级最高，触发后尽快同步处理，保证响应性。

---

## 原生事件 vs React 事件的执行顺序

监听器的位置决定执行顺序：

```
document
  └── #root（React 监听器在这里）
        └── #btn（原生监听器绑在这里）
```

click 从上往下 capture，再从下往上 bubble：

```
capture 阶段：
  root capture → React 执行 onClickCapture

bubble 阶段：
  btn bubble  → 原生监听器执行        ← 先
  root bubble → React 执行 onClick    ← 后
```

**子节点上的原生事件比 React bubble 早执行**，不是特殊机制，就是 DOM 冒泡的自然顺序——btn 比 root 先冒泡。

如果原生监听器也绑在 root 上，则取决于注册顺序，React 在 `createRoot` 时注册，后来添加的原生监听器会在 React 之后执行。

---

## 事件插件

不同类型事件由不同插件处理：

| 插件 | 负责事件 |
|---|---|
| SimpleEventPlugin | 绝大多数事件（click、keydown…） |
| EnterLeaveEventPlugin | mouseenter / mouseleave（不冒泡，模拟实现） |
| ChangeEventPlugin | onChange（跨浏览器兼容） |
| SelectEventPlugin | onSelect |
| BeforeInputEventPlugin | onBeforeInput |
