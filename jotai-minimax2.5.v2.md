# Jotai 实现原理深入分析

> Jotai 是一个 React 状态管理库，核心设计理念是原子化状态管理。
> 版本：2.18.0 | 核心代码：~1000 行

---

## 一、核心概念

### 1.1 Atom（原子）

Atom 是状态的基本单元，本质上是一个**配置对象**：

```typescript
// src/vanilla/atom.ts:102-125
export function atom<Value, Args extends unknown[], Result>(
  read?: Value | Read<Value, SetAtom<Args, unknown>>,
  write?: Write<Args, Result>,
) {
  const key = `atom${++keyCount}`  // 全局唯一标识
  const config = {
    toString() { /* ... */ },
  } as WritableAtom<Value, Args, Result> & { init?: Value }

  if (typeof read === 'function') {
    config.read = read  // 派生 atom
  } else {
    config.init = read  // 原始 atom 的初始值
    config.read = defaultRead  // 默认读取逻辑
    config.write = defaultWrite
  }
  if (write) {
    config.write = write
  }
  return config
}
```

Atom 的三种形态：

```typescript
// 原始原子（Primitive Atom）
const countAtom = atom(0)

// 派生原子（Derived Atom）- 只读
const doubledAtom = atom((get) => get(countAtom) * 2)

// 可写原子（Writable Atom）
const incrementAtom = atom(
  (get) => get(countAtom),                          // read
  (get, set, n = 1) => set(countAtom, get(countAtom) + n)  // write
)
```

**关键点**：
- `key` 是全局递增的数字，用于调试和追踪
- `read` 函数接收 `get` 参数，可以读取其他 atom
- `write` 函数接收 `get` 和 `set`，可以读取和修改其他 atom

### 1.2 Store（存储）

Store 是原子的容器，管理所有原子的状态和依赖关系：

```typescript
import { createStore } from 'jotai/vanilla'

const store = createStore()
store.get(countAtom)    // 读取
store.set(countAtom, 5) // 写入
store.sub(atom, listener) // 订阅变化
```

---

## 二、核心数据结构

### 2.1 AtomState（原子状态）

```typescript
// src/vanilla/internals.ts:22-41
type AtomState<Value = AnyValue> = {
  // 依赖关系: atom -> epoch 版本的映射
  // 记录这个 atom 依赖哪些 atoms 及其版本号
  readonly d: Map<AnyAtom, EpochNumber>

  // 等待中的 Promise: 记录哪些 atom 依赖这个 atom 的异步值
  // 当 atom 值为 Promise 时，依赖它的 atoms 需要等待
  readonly p: Set<AnyAtom>

  // 版本号 (Epoch)，每次状态变化递增
  // 用于判断是否需要重新计算，无需深比较
  n: EpochNumber

  // 缓存的值
  v?: Value

  // 错误状态
  e?: AnyError
}
```

### 2.2 Mounted（挂载状态）

```typescript
// src/vanilla/internals.ts:49-58
type Mounted = {
  // 监听器集合: 组件通过 useAtomValue 注册的回调
  readonly l: Set<() => void>

  // 依赖的 atoms: 这个 atom 依赖哪些 atoms
  readonly d: Set<AnyAtom>

  // 依赖这个 atom 的 atoms（反向追踪）
  // 用于传播失效通知
  readonly t: Set<AnyAtom>

  // onMount 清理函数
  u?: () => void
}
```

### 2.3 Store 内部存储

```typescript
// src/vanilla/internals.ts:1005-1059
const store = {
  get(atom) { /* ... */ },
  set(atom, ...args) { /* ... */ },
  sub(atom, listener) { /* ... */ },
}

// 使用 WeakMap 存储，atom 没有引用时自动 GC
const atomStateMap = new WeakMap<AnyAtom, AtomState>()
const mountedMap = new WeakMap<AnyAtom, Mounted>()
const invalidatedAtoms = new WeakMap<AnyAtom, EpochNumber>()
const changedAtoms = new Set<AnyAtom>()
```

---

## 三、版本号（Epoch）机制

### 3.1 为什么用版本号？

```typescript
// 值比较的问题
const prev = { name: 'john' }
const next = { name: 'john' }
prev === next  // false！对象永远不相等

// 版本号的解决方案
atomState.n++  // 只需自增一个数字
```

### 3.2 版本号的递增

```typescript
// src/vanilla/internals.ts:896-924
const BUILDING_BLOCK_setAtomStateValueOrPromise = (...) => {
  const atomState = ensureAtomState(store, atom)
  const hasPrevValue = 'v' in atomState
  const prevValue = atomState.v

  atomState.v = valueOrPromise
  delete atomState.e

  // 只有值真正变化时才递增版本号
  if (!hasPrevValue || !Object.is(prevValue, atomState.v)) {
    ++atomState.n  // 版本号自增
    if (isPromiseLike(prevValue)) {
      abortPromise(store, prevValue)
    }
  }
}
```

### 3.3 判断是否需要重新计算

```typescript
// src/vanilla/internals.ts:520-540
if (isAtomStateInitialized(atomState)) {
  // 已挂载且未失效的 atom，可以直接用缓存
  if (mountedMap.has(atom) && invalidatedAtoms.get(atom) !== atomState.n) {
    return atomState
  }

  // 检查所有依赖的版本号是否变化
  let hasChangedDeps = false
  for (const [a, n] of atomState.d) {
    if (readAtomState(store, a).n !== n) {  // 版本号比较！
      hasChangedDeps = true
      break
    }
  }

  // 依赖没变，用缓存
  if (!hasChangedDeps) {
    return atomState
  }
}
```

---

## 四、依赖追踪机制

这是 jotai 最精妙的设计，完全在 vanilla 层实现，无需 React 的 Context。

### 4.1 读取时的依赖收集

```typescript
// src/vanilla/internals.ts:563-593
const getter = <V>(a: Atom<V>) => {
  if (a === (atom as AnyAtom)) {
    // 避免自我引用
    return returnAtomValue(ensureAtomState(store, a))
  }

  // 读取目标 atom 的状态
  const aState = readAtomState(store, a)
  try {
    return returnAtomValue(aState)
  } finally {
    // 关键：记录依赖关系和版本号
    nextDeps.set(a, aState.n)       // 记录我依赖了这个 atom 及其版本
    atomState.d.set(a, aState.n)    // 同时更新 atomState

    // 如果是 promise，追踪它
    if (isPromiseLike(atomState.v)) {
      addPendingPromiseToDependency(atom, atomState.v, aState)
    }

    // 更新反向依赖：a 现在被 atom 依赖
    if (mountedMap.has(atom)) {
      mountedMap.get(a)?.t.add(atom)
    }
  }
}
```

### 4.2 依赖追踪流程

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

## 五、写入与更新传播

### 5.1 写入流程

```typescript
// src/vanilla/internals.ts:698-757
const BUILDING_BLOCK_writeAtomState = (store, atom, ...args) => {
  const setter: Setter = <V, As, R>(a, ...args) => {
    const aState = ensureAtomState(store, a)
    try {
      if (a === atom) {
        // 写入自身（primitive atom）
        const prevEpochNumber = aState.n
        setAtomStateValueOrPromise(store, a, v)
        mountDependencies(store, a)

        // 版本号变化了
        if (prevEpochNumber !== aState.n) {
          changedAtoms.add(a)
          invalidateDependents(store, a)  // 失效所有依赖的 atoms
          storeHooks.c?.(a)
        }
        return undefined
      } else {
        // 写入其他 atom（调用该 atom 的 write 函数）
        return writeAtomState(store, a, ...args)
      }
    } finally {
      if (!isSync) {
        recomputeInvalidatedAtoms(store)  // 重新计算
        flushCallbacks(store)              // 通知订阅者
      }
    }
  }

  return atomWrite(store, atom, getter, setter, ...args)
}
```

### 5.2 失效传播（DFS）

```typescript
// src/vanilla/internals.ts:676-696
const BUILDING_BLOCK_invalidateDependents = (store, atom) => {
  const stack: AnyAtom[] = [atom]
  while (stack.length) {
    const a = stack.pop()!
    const aState = ensureAtomState(store, a)

    // 遍历所有依赖这个 atom 的 atoms
    for (const d of getMountedOrPendingDependents(a, aState, mountedMap)) {
      const dState = ensureAtomState(store, d)

      // 如果还没失效，标记失效并继续传播
      if (invalidatedAtoms.get(d) !== dState.n) {
        invalidatedAtoms.set(d, dState.n)
        stack.push(d)
      }
    }
  }
}
```

### 5.3 拓扑排序重新计算

这是最复杂的部分，使用 DFS 实现拓扑排序：

```typescript
// src/vanilla/internals.ts:428-499
const BUILDING_BLOCK_recomputeInvalidatedAtoms = (store) => {
  // 步骤 1: 拓扑排序
  const topSortedReversed: [atom: AnyAtom, atomState: AtomState][] = []
  const visiting = new WeakSet<AnyAtom>()
  const visited = new WeakSet<AnyAtom>()
  const stack: AnyAtom[] = Array.from(changedAtoms)

  while (stack.length) {
    const a = stack[stack.length - 1]!
    const aState = ensureAtomState(store, a)

    if (visited.has(a)) {
      stack.pop()
      continue
    }

    if (visiting.has(a)) {
      // 发现已访问的节点，加入排序结果
      if (invalidatedAtoms.get(a) === aState.n) {
        topSortedReversed.push([a, aState])
      }
      visited.add(a)
      stack.pop()
      continue
    }

    visiting.add(a)
    // 将未访问的依赖压入栈
    for (const d of getMountedOrPendingDependents(a, aState, mountedMap)) {
      if (!visiting.has(d)) {
        stack.push(d)
      }
    }
  }

  // 步骤 2: 反向遍历，从叶子节点开始重新计算
  for (let i = topSortedReversed.length - 1; i >= 0; --i) {
    const [a, aState] = topSortedReversed[i]!

    // 检查是否有依赖变化
    let hasChangedDeps = false
    for (const dep of aState.d.keys()) {
      if (changedAtoms.has(dep)) {
        hasChangedDeps = true
        break
      }
    }

    // 有变化才重新计算
    if (hasChangedDeps) {
      readAtomState(store, a)
      mountDependencies(store, a)
    }
  }
}
```

**为什么需要拓扑排序？**
- 确保依赖顺序正确：先计算被依赖的 atom，再计算依赖它的 atom
- 例如：`c = a + b`，必须先算 `a` 和 `b`，再算 `c`

---

## 六、React 集成层

### 6.1 useAtomValue 的订阅机制

```typescript
// src/react/useAtomValue.ts:119-183
export function useAtomValue<Value>(atom: Atom<Value>, options?: Options) {
  const store = useStore(options)

  // 使用 useReducer 而非 useState
  // 因为需要在回调中更新状态
  const [[valueFromReducer, storeFromReducer, atomFromReducer], rerender] =
    useReducer((prev) => {
      const nextValue = store.get(atom)
      // 快速路径：值没变就不触发重渲染
      if (
        Object.is(prev[0], nextValue) &&
        prev[1] === store &&
        prev[2] === atom
      ) {
        return prev
      }
      return [nextValue, store, atom]
    }, undefined,
    () => [store.get(atom), store, atom])

  // 订阅变化
  useEffect(() => {
    const unsub = store.sub(atom, () => rerender())
    // 立即触发一次 rerender 确保同步
    rerender()
    return unsub
  }, [store, atom, delay, promiseStatus])

  // 处理异步值
  if (isPromiseLike(value)) {
    const promise = createContinuablePromise(store, value, () => store.get(atom))
    return use(promise)  // React 19 的 use hook
  }
  return value
}
```

### 6.2 useReducer vs useState

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

### 6.3 store.sub 的实现

```typescript
// src/vanilla/internals.ts:944-958
const BUILDING_BLOCK_storeSub = (store, atom, listener) => {
  const mounted = mountAtom(store, atom)  // 确保 atom 被挂载
  const listeners = mounted.l
  listeners.add(listener)

  flushCallbacks(store)  // 立即 flush 确保同步

  return () => {
    listeners.delete(listener)
    unmountAtom(store, atom)
    flushCallbacks(store)
  }
}
```

---

## 七、异步支持

### 7.1 Promise 追踪

```typescript
// src/vanilla/internals.ts:252-262
function addPendingPromiseToDependency(
  atom: AnyAtom,
  promise: PromiseLike<AnyValue>,
  dependencyAtomState: AtomState,
): void {
  if (!dependencyAtomState.p.has(atom)) {
    dependencyAtomState.p.add(atom)
    // promise settle 后清理
    const cleanup = () => dependencyAtomState.p.delete(atom)
    promise.then(cleanup, cleanup)
  }
}
```

当一个 atom 的值是 Promise 时：
- 将该 atom 添加到依赖的 `p` 集合
- 当 promise 完成时，清理引用
- `getMountedOrPendingDependents` 会返回有 pending promise 的 atoms

### 7.2 Continuation Promise

```typescript
// src/react/useAtomValue.ts:60-102
const createContinuablePromise = <T>(store, promise, getValue) => {
  return new Promise<T>((resolve, reject) => {
    const onAbort = () => {
      // 获取最新值（可能再次是 promise）
      const nextValue = getValue()
      if (isPromiseLike(nextValue)) {
        // 继续等待新的 promise
        nextValue.then(onFulfilled(nextValue), onRejected(nextValue))
        registerAbortHandler(store, nextValue, onAbort)
      } else {
        resolve(nextValue)
      }
    }

    promise.then(onFulfilled(promise), onRejected(promise))
    registerAbortHandler(store, promise, onAbort)
  })
}
```

这实现了 **Promise 串联**：当 promise A 完成后，自动获取并等待新的 promise。

---

## 八、内存管理与清理

### 8.1 WeakMap 的使用

```typescript
const atomStateMap = new WeakMap()    // atom → AtomState
const mountedMap = new WeakMap()      // atom → Mounted
const invalidatedAtoms = new WeakMap() // atom → EpochNumber
```

使用 `WeakMap` 的好处：
- 当 atom 没有其他引用时，可以被垃圾回收
- 对应的 `AtomState` 也会被回收
- 不需要手动清理内存

### 8.2 挂载/卸载逻辑

```typescript
// src/vanilla/internals.ts:859-894
const BUILDING_BLOCK_unmountAtom = (store, atom) => {
  const mounted = mountedMap.get(atom)
  if (!mounted || mounted.l.size) {
    return mounted  // 仍有订阅者，不卸载
  }

  // 检查是否有其他 atom 依赖这个 atom
  let isDependent = false
  for (const a of mounted.t) {
    if (mountedMap.get(a)?.d.has(atom)) {
      isDependent = true
      break
    }
  }

  if (!isDependent) {
    // 卸载自己
    mountedMap.delete(atom)
    // 卸载依赖
    for (const a of atomState.d.keys()) {
      unmountAtom(store, a)
    }
  }
}
```

---

## 九、完整更新流程图

```
用户调用 setAtom(newValue)
         │
         ▼
┌─────────────────────────────────────────────────────┐
│ writeAtomState                                      │
│  - setAtomStateValueOrPromise(newValue)             │
│    - atomState.v = newValue                         │
│    - atomState.n++  (版本号自增)                     │
│  - changedAtoms.add(atom)                           │
│  - invalidateDependents(atom)                        │
└─────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│ invalidateDependents (DFS 传播失效)                   │
│  - 遍历所有依赖 atom 的 atoms                        │
│  - 标记 invalidatedAtoms.set(dep, depState.n)       │
│  - 继续传播给它们的依赖                               │
└─────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│ flushCallbacks                                      │
│  - 收集所有 changedAtoms 的 listeners               │
│  - 通知 React 组件 rerender                         │
└─────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│ React 组件重渲染                                     │
│  - useAtomValue 再次调用 store.get(atom)            │
│  - readAtomState 检查依赖版本号                     │
│  - 发现版本号已变，调用 atom.read 重新计算           │
│  - 返回新值，触发组件更新                           │
└─────────────────────────────────────────────────────┘
```

---

## 十、分层架构

```
┌─────────────────────────────────────────────────────┐
│                    React 组件                         │
│  useAtom(atom) → [value, setValue]                  │
└─────────────────────┬───────────────────────────────┘
                      │
        ┌─────────────▼─────────────┐
        │      useAtomValue        │
        │   useReducer + useEffect │
        └─────────────┬─────────────┘
                      │
        ┌─────────────▼─────────────┐
        │       Store (vanilla)     │
        │  ┌─────────────────────┐ │
        │  │ atomStateMap        │ │
        │  │ mountedMap          │ │
        │  │ invalidatedAtoms    │ │
        │  │ changedAtoms        │ │
        │  └─────────────────────┘ │
        └─────────────┬─────────────┘
                      │
        ┌─────────────▼─────────────┐
        │        Atoms              │
        │  - Primitive: atom(0)    │
        │  - Derived: atom(get=>..)│
        │  - Writable: atom(r, w)  │
        └──────────────────────────┘
```

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

## 十一、与 Redux/Zustand 的对比

| 特性 | Jotai | Redux | Zustand |
|------|-------|-------|---------|
| 状态粒度 | Atom 级别 | Global store | Store 级别 |
| 重渲染 | 精确订阅 | 整棵组件树 | 可选择精确 |
| 派生状态 | 原生支持 | Selector | Selector |
| 异步 | 原生支持 | Middleware | Middleware |
| 核心大小 | ~3KB | ~7KB | ~1KB |
| 依赖方式 | 运行时自动 | 手动选择器 | 手动选择器 |

---

## 十二、设计优势

- **自动依赖追踪**：无需手动管理依赖关系，库会自动计算最小更新范围
- **无字符串键**：atom 是引用类型，避免键名冲突
- **细粒度更新**：只更新依赖的组件
- **Vanilla 核心**：状态逻辑与 React 解耦，便于测试和扩展
- **强大的工具函数**：atomWithStorage、atomFamily、selectAtom 等
- **极简核心**：约 3KB，核心代码约 1000 行
- **TypeScript 友好**：完整类型支持
- **Promise 原生支持**：内置异步状态管理

这种设计使得 Jotai 实现了零成本抽象——核心只有约 1000 行代码，却提供了完整的状态管理能力。
