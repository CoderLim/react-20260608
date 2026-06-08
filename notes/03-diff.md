# Diff 算法

源码：`packages/react-reconciler/src/ReactChildFiber.js`

## 为什么需要 Diff？

每次状态更新，React 重新执行组件函数生成新 JSX 树。如果直接把 DOM 全删重建，性能很差。

Diff 的目标：**对比新旧 Fiber 树，找出最小变更**，尽量复用已有 DOM 节点。

---

## 单节点 Diff

新节点是单个 Element，遍历旧兄弟节点找复用目标：

```
key 不匹配 → 删掉这个旧节点，继续找下一个兄弟
key 匹配，type 不匹配 → 把这个及后面所有兄弟全删，创建新节点
key 匹配，type 匹配 → 复用，其余兄弟删掉
```

**key 匹配时为什么要全删？**
key 匹配意味着"目标已定位"，key 唯一，后面不可能再有更好的候选。
最终只保留一个子节点，所以其余全删。

---

## 多节点 Diff：两轮遍历

### 第一轮：验证顺序是否稳定

从左到右同时遍历新旧节点，**核心问题是"顺序还对得上吗"**：

- `updateSlot` 先比 key：
  - **key 不匹配 → 返回 null → break**（顺序乱了，进第二轮）
  - key 匹配 → 再交给 `updateElement` 比 type：
    - type 匹配 → 复用
    - type 不匹配 → 创建新节点，**不 break**（顺序还在，继续）

```
旧：A  B  C  D
新：A  B  X  D      （C 换成了 X，key 不同）

位置0：key 匹配，type 匹配 → 复用 A
位置1：key 匹配，type 匹配 → 复用 B
位置2：key 不匹配 → break，进第二轮
```

```
旧：A  B(type=Btn)  C
新：A  B(type=Link) C    （同 key，换了 type）

位置0：key 匹配，type 匹配 → 复用 A
位置1：key 匹配，type 不匹配 → 旧 B 丢弃，创建新 B，不 break
位置2：key 匹配，type 匹配 → 复用 C
```

**第一轮 break 的唯一条件是 key 不匹配。type 不匹配不中断，因为它不影响顺序判断。**

这与单节点 Diff 不同——单节点找的是"唯一目标"，key 匹配即定位完成；
数组第一轮验证的是"顺序"，type 不影响顺序，所以更宽松。

特殊情况：新节点先遍历完 → 删余下旧节点，直接结束，不进第二轮。

---

### 第二轮：Map 精确匹配

第一轮 break 后，剩余旧节点存入 Map，再遍历剩余新节点查找：

```
有 key → key 作为 Map 的键
无 key → index 作为 Map 的键
```

- 找到 → 复用，从 Map 删除
- 没找到 → 创建新节点
- Map 中剩余 → 全部删除

**例子：**

```
旧：A  B  C  D
新：A  C  E  B

第一轮：复用 A，然后 B vs C key 不匹配，break

第二轮：
  Map = { B→fiber, C→fiber, D→fiber }

  新节点 C → 找到 → 复用
  新节点 E → 没找到 → 新建
  新节点 B → 找到 → 复用

  Map 剩余 D → 删除
```

---

## 移动判断：lastPlacedIndex

复用不等于不用移动 DOM。用 `lastPlacedIndex` 记录已安置旧节点的最大 index：

```
旧节点的原 index >= lastPlacedIndex → 不需要移动，更新 lastPlacedIndex
旧节点的原 index <  lastPlacedIndex → 需要移动（原来在前，现在要放后面）
```

**例子：**

```
旧：A(0)  B(1)  C(2)  D(3)
新：A     C     E     B

处理 A：oldIndex=0 >= 0 → 不动，lastPlacedIndex=0
处理 C：oldIndex=2 >= 0 → 不动，lastPlacedIndex=2
处理 E：新节点 → 插入
处理 B：oldIndex=1 < 2  → 需要移动

最终 DOM 操作：移动 B，插入 E，删除 D
```

---

## 没有 Key 时

没有 key，所有节点 key 为 `null`，`null === null` 始终成立，**key 这关对所有节点直接放行**。

**结果：退化为按位置 + type 比较。**

- 第一轮：null===null 永远不 break，完全靠 type 决定能否复用
- 第二轮 Map：用 index 作键，本质还是按位置匹配

```
旧（无key）：<A/>  <B/>  <C/>
新（无key）：<B/>  <A/>  <C/>

位置0：null===null → type A≠B → 创建新 B，旧 A 丢弃
位置1：null===null → type B≠A → 创建新 A，旧 B 丢弃
位置2：null===null → type C=C → 复用 C
```

明明只是顺序换了，却创建了两个新节点。**这是无 key 列表的性能代价。**

加了 key，React 才能通过第二轮 Map 找到跨位置的匹配，实现移动而非重建。

---

## 总结

```
新子节点
  │
  ├─ 单个 → 遍历旧兄弟，按 key + type 找唯一目标
  │         key 匹配即定位完成，其余全删
  │
  └─ 数组 → 两轮遍历
        │
        第一轮：验证顺序
          key 不匹配 → break（顺序乱了）
          key 匹配，type 不匹配 → 新建，继续（顺序没乱）
          key 匹配，type 匹配 → 复用，继续
          新节点遍历完 → 删余下旧节点，结束
        │
        第二轮：Map 匹配剩余节点
          有 key → 用 key 查；无 key → 用 index 查
          找到 → 复用；没找到 → 新建；剩余 → 删除
        │
        每次复用：lastPlacedIndex 判断是否需要移动 DOM

没有 key → 退化为按位置匹配，顺序变动 = 重建，性能差
```
