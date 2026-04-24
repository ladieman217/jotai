# Jotai 实现原理深入分析

> Jotai 是一个 React 状态管理库，核心设计理念是原子化状态管理。

## 核心概念

### 1. Atom（原子）

Atom 是状态的基本单元，类似于 `useState`，但更灵活：

```typescript
// 原始原子
const countAtom = atom(0)

// 派生原子（只读）
const doubledAtom = atom((get) => get(countAtom) * 2)

// 派生原子（可写）
const incrementAtom = atom(
  (get) => get(countAtom),
  (get, set, newValue) => set(countAtom, newValue)
)
```

### 2. Store（存储）

Store 是原子的容器，管理所有原子的状态和依赖关系：

```typescript
import { createStore } from 'jotai'

const store = createStore()
store.get(countAtom)    // 读取
store.set(countAtom, 5) // 写入
```

---

## 核心数据结构

### AtomState（原子状态）

```typescript
type AtomState<Value> = {
  // 依赖关系: atom -> epoch 版本的映射
  readonly d: Map<AnyAtom, EpochNumber>
  // 等待中的 Promise 依赖
  readonly p: Set<AnyAtom>
  // 版本号 (Epoch)，每次状态变化递增
  n: EpochNumber
  // 缓存的值
  v?: Value
  // 错误状态
  e?: Error
}
```

### Mounted（挂载状态）

```typescript
type Mounted = {
  // 监听器集合，值变化时通知
  readonly l: Set<() => void>
  // 当前 atom 依赖的其他 atoms
  readonly d: Set<AnyAtom>
  // 依赖当前 atom 的其他 atoms
  readonly t: Set<AnyAtom>
  // 卸载时的回调
  u?: () => void
}
```

---

## 版本号（Epoch）机制

Jotai 使用版本号而非直接比较值来实现细粒度更新：

```
初始: countAtom.n = 0
第一次 set(1): countAtom.n = 1
第二次 set(2): countAtom.n = 2
```

读取派生 atom 时，保存依赖的版本号：

```typescript
// 读取时记录依赖版本
atomState.d.set(dependencyAtom, dependencyAtomState.n)
```

**判断是否需要重新计算**：

```typescript
// 比较依赖版本号
for (const [a, n] of atomState.d) {
  if (readAtomState(store, a).n !== n) {  // 依赖已更新
    hasChangedDeps = true
    break
  }
}
```

---

## 依赖追踪流程

当读取一个派生 atom 时：

```
1. readAtomState(store, atom)
   ↓
2. 调用 atom.read(getter, options)
   ↓
3. getter(otherAtom) 触发读取
   ↓
4. 递归调用 readAtomState(store, otherAtom)
   ↓
5. 记录依赖: atomState.d.set(otherAtom, otherAtom.n)
   ↓
6. 返回值并缓存
```

---

## 变更传播算法

当一个 atom 被写入时：

```typescript
// store.set(countAtom, 5) 调用链

1. writeAtomState(store, atom, value)
   ↓
2. setAtomStateValueOrPromise()  // 设置新值，增加版本号
   ↓
3. changedAtoms.add(atom)        // 标记已改变
   ↓
4. invalidateDependents()        // 使依赖的 atoms 无效
   ↓
5. flushCallbacks()              // 执行回调
```

### 拓扑排序重计算

`recomputeInvalidatedAtoms` 使用 DFS 拓扑排序：

```typescript
// 简化逻辑
const topSortedReversed: AnyAtom[] = []

function dfs(atom) {
  if (visited.has(atom)) return
  visiting.add(atom)

  // 先访问所有依赖
  for (const dep of getDependents(atom)) {
    dfs(dep)
  }

  // 依赖都已处理，加入列表
  topSortedReversed.push(atom)
  visited.add(atom)
}

// 从后向前遍历，保证依赖先于派生者更新
for (let i = topSortedReversed.length - 1; i >= 0; i--) {
  const atom = topSortedReversed[i]
  if (hasChangedDeps(atom)) {
    readAtomState(store, atom)  // 重新计算
  }
}
```

---

## React 集成层

### useAtomValue 实现

```typescript
function useAtomValue(atom, options) {
  // 1. 使用 useReducer 创建订阅机制
  const [state, rerender] = useReducer(reducer, initialState)

  // 2. 初始化读取
  const value = store.get(atom)

  // 3. 订阅变更
  useEffect(() => {
    const unsub = store.sub(atom, () => {
      rerender()  // 触发重渲染
    })
    return unsub
  }, [store, atom])

  // 4. 处理异步 atom
  if (isPromiseLike(value)) {
    return use(promise)  // React 19+ 或自定义实现
  }

  return value
}
```

### useReducer vs useState

Jotai 选择 `useReducer` 而非 `useState` 的原因：

1. **避免无限循环**：reducer 可以返回相同引用而不触发更新
2. **更灵活的依赖检测**：可以在 reducer 中比较 atom 值
3. **更好的引用相等性控制**

```typescript
// useReducer 可以这样控制更新
if (Object.is(prevValue, nextValue)) {
  return prev  // 相同引用，不触发更新
}
```

---

## 内存管理

### WeakMap 存储

```typescript
const atomStateMap = new WeakMap<AnyAtom, AtomState>()
const mountedMap = new WeakMap<AnyAtom, Mounted>()
```

使用 `WeakMap` 的好处：

- atom 没有其他引用时自动垃圾回收
- 不需要手动清理内存

### 自动卸载

```typescript
// unmountAtom 检查是否还有其他依赖
function unmountAtom(store, atom) {
  const mounted = mountedMap.get(atom)

  // 如果没有监听器且没有依赖它的 atoms
  if (!mounted.l.size && !hasDependents(atom)) {
    mountedMap.delete(atom)
    // 递归卸载依赖
    for (const dep of atomState.d.keys()) {
      unmountAtom(store, dep)
    }
  }
}
```

---

## 异步支持

### Promise 处理

```typescript
// 注册 abort handler
registerAbortHandler(promise, () => controller.abort())

// 依赖追踪
if (isPendingPromise(atomState.v)) {
  addPendingPromiseToDependency(atom, promise, dependencyState)
}

// 可取消的读取
const controller = new AbortController()
const value = await atom.read(get, { signal: controller.signal })
```

### Continuation Promise

```typescript
// 创建可继续的 Promise，支持重新获取
function createContinuablePromise(promise, getValue) {
  return new Promise((resolve, reject) => {
    const onAbort = () => {
      const nextValue = getValue()  // 重新获取最新值
      if (isPromiseLike(nextValue)) {
        // 继续等待新的 promise
        nextValue.then(resolve, reject)
      } else {
        resolve(nextValue)
      }
    }
    promise.then(..., onAbort)  // 原始完成或取消时触发
  })
}
```

---

## 完整数据流图

```
┌─────────────────────────────────────────────────────────────────┐
│                         React 组件                               │
│  const [count, setCount] = useAtom(countAtom)                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    useAtomValue / useSetAtom                    │
│  - store.get(atom) 读取                                         │
│  - store.set(atom, value) 写入                                   │
│  - store.sub(atom, listener) 订阅                                │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Store API                                 │
│  get / set / sub                                                │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     核心实现 (internals.ts)                      │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │
│  │ atomStateMap│  │ mountedMap   │  │ 回调队列    │           │
│  │ (WeakMap)   │  │ (WeakMap)    │  │             │           │
│  └─────────────┘  └─────────────┘  └─────────────┘           │
│                                                                 │
│  核心函数:                                                      │
│  - readAtomState: 读取并计算 atom 值                            │
│  - writeAtomState: 写入并传播变更                               │
│  - invalidateDependents: 标记依赖无效                          │
│  - recomputeInvalidatedAtoms: 拓扑排序重计算                    │
│  - flushCallbacks: 执行监听器回调                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 关键技术点总结

| 特性 | 实现方式 |
|------|----------|
| 依赖追踪 | 在 getter 中记录 `atomState.d.set(dep, dep.n)` |
| 变更检测 | 比较版本号 `atomState.n` |
| 细粒度更新 | 只重计算受影响的派生 atoms |
| 异步支持 | Promise + AbortController |
| 内存管理 | WeakMap + 自动卸载逻辑 |
| React 集成 | useReducer + 订阅机制 |

---

## 分层架构

```
┌─────────────────────────────────────┐
│         React 层 (src/react)        │  ← useAtom, Provider
├─────────────────────────────────────┤
│       Vanilla 层 (src/vanilla)       │  ← 核心逻辑，无框架依赖
├─────────────────────────────────────┤
│   atom.ts  │  store.ts  │ internals.ts │
└─────────────────────────────────────┘
```

---

## 设计优势

- **无字符串键**：atom 是引用类型，避免键名冲突
- **细粒度更新**：只更新依赖的组件
- **Vanilla 核心**：状态逻辑与 React 解耦，便于测试
- **强大的工具函数**：atomWithStorage、atomFamily、selectAtom 等
- **极简核心**：约 2kb 的核心代码
- **TypeScript 友好**：完整类型支持

这种设计使得 Jotai 实现了零成本抽象——核心只有约 1000 行代码，却提供了完整的状态管理能力。
