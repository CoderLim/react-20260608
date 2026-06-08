# Diff 算法

源码：`packages/react-reconciler/src/ReactChildFiber.js`

## 为什么需要 Diff？

每次状态更新，React 重新执行组件函数生成新 JSX 树。如果直接把 DOM 全删重建，性能很差。

Diff 的目标：**对比新旧 Fiber 树，找出最小变更**，尽量复用已有 DOM 节点。

---

## 两种情况

### 单节点 Diff

新节点是单个 Element，去旧的兄弟节点里找能不能复用：

```
遍历旧兄弟：
  key 不匹配 → 删掉这个旧节点，继续找下一个兄弟
  key 匹配，type 不匹配 → 把这个及后面所有兄弟全删，创建新节点
  key 和 type 都匹配 → 复用，其余兄弟删掉
```

**key 决定找哪个，type 决定能不能复用。**

### 多节点 Diff

新节点是数组，走两轮遍历。

---

## 多节点 Diff：两轮遍历

### 为什么两轮？

列表变更有两类情况：
1. **顺序没变**，只是内容更新 → 按位置一一对应，很快
2. **顺序变了**，节点移动/插入/删除 → 需要用 key 精确匹配

两轮设计：先用简单方式处理第 1 类，处理不了再上第 2 类。

---

### 第一轮：按位置对比

同时从左到右遍历新旧节点，逐位置尝试复用：

```
旧：A  B  C  D
新：A  B  X  D

位置0：A vs A → key 匹配，type 匹配 → 复用
位置1：B vs B → key 匹配，type 匹配 → 复用
位置2：C vs X → key 不匹配 → break，进入第二轮
```

**一旦 key 不匹配就停下来**，不再按位置比。

特殊情况：新节点先遍历完，不用进第二轮：

```
旧：A  B  C  D
新：A  B

→ 复用 A、B，删掉旧的 C、D，结束
```

---

### 第二轮：Map 精确匹配

第一轮 break 后，新旧各有剩余节点需要处理。

**第一步**：剩余旧节点存入 Map
- 有 key → `key → Fiber`
- 无 key → `index → Fiber`

**第二步**：遍历剩余新节点，去 Map 里查
- 找到 → 复用，从 Map 删掉这条记录
- 没找到 → 创建新节点

**第三步**：Map 里还剩的旧节点 → 全部删除

**例子：**

```
旧：A  B  C  D
新：A  C  E  B

第一轮：复用 A，然后 B vs C key 不匹配，break

第二轮：
  剩余旧节点 B C D → Map = { B→fiber, C→fiber, D→fiber }

  新节点 C → Map 里有 → 复用
  新节点 E → Map 里没有 → 新建
  新节点 B → Map 里有 → 复用

  Map 剩余 D → 删除
```

---

## 移动判断：lastPlacedIndex

复用不等于不用移动 DOM。React 用 `lastPlacedIndex` 来判断。

**含义**：已安置好的旧节点的最大 index。

```
复用一个旧节点时：
  旧节点的原 index >= lastPlacedIndex → 不需要移动，更新 lastPlacedIndex
  旧节点的原 index <  lastPlacedIndex → 需要移动（原来在前，现在要放后面）
```

**例子：**

```
旧：A(0)  B(1)  C(2)  D(3)
新：A     C     E     B

处理 A：oldIndex=0，lastPlacedIndex=0 → 不动，lastPlacedIndex=0

第二轮：
  C：oldIndex=2，lastPlacedIndex=0 → 2≥0，不动，lastPlacedIndex=2
  E：新节点 → 插入
  B：oldIndex=1，lastPlacedIndex=2 → 1<2，需要移动

最终 DOM 操作：移动 B，插入 E，删除 D
```

---

## Key 的本质作用

没有 key，React 只能按位置对应，顺序一变就认不出来：

```
旧（无key）：<Input value="hello"/>  <Input value="world"/>
新（头部插入）：<Input value="new"/>  <Input value="hello"/>  <Input value="world"/>

第一轮按位置：
  位置0：两个 Input type 相同 → 复用，把 "hello" 的 fiber 用来渲染 "new"
  位置1：两个 Input type 相同 → 复用，把 "world" 的 fiber 用来渲染 "hello"
  位置2：无旧节点 → 新建
```

Input 有内部状态（焦点、光标等），这样复用会导致状态错乱。**加 key 让 React 按身份而非位置追踪节点。**

---

## 总结

```
新子节点
  │
  ├─ 单个 → 遍历旧兄弟，按 key + type 找匹配
  │
  └─ 数组 → 两轮遍历
        │
        第一轮：按位置逐个对比
          命中 → 复用，继续
          未命中 → break 进第二轮
          新节点用完 → 删余下旧节点，结束
        │
        第二轮：旧节点存 Map，新节点逐个查 Map
          找到 → 复用；找不到 → 新建；Map 剩余 → 删除
        │
        每次复用：用 lastPlacedIndex 判断是否需要移动 DOM
```
