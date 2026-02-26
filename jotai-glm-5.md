# Jotai 实现原理（深入版）

## 1. 核心概念

Jotai 是一个**原子化** 状态管理库，核心理念是"自下而上"的状态管理——从最小的状态单元开始组合。与 Redux 的"单一 Store"模式不同，Jotai 允许创建任意数量的原子状态，并通过依赖图自动管理它们之间的关系。

## 2. 核心架构

```
┌─────────────────────────────────────────────────────────────┐
│                    React Layer                               │
│  ┌─────────────┐  ┌───────────────┐  ┌──────────────────┐   │
│  │ Provider    │  │ useAtomValue  │  │ useSetAtom       │   │
│  │ (Context)   │  │ (订阅+读取)   │  │ (写入)           │   │
│  └──────┬──────┘  └───────┬───────┘  └────────┬─────────┘   │
│         │                 │                    │             │
│         └─────────────────┼────────────────────┘             │
│                           ▼                                  │
├─────────────────────────────────────────────────────────────┤
│                    Store Layer (Vanilla)                     │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    Store                                 ││
│  │  get(atom) → value                                      ││
│  │  set(atom, value) → void                                ││
│  │  sub(atom, listener) → unsubscribe                      ││
│  └──────────────────────┬──────────────────────────────────┘│
│                         │                                    │
│  ┌──────────────────────▼──────────────────────────────────┐│
│  │              Building Blocks ( internals.ts )           ││
│  │  ┌─────────────────┐  ┌─────────────────┐               ││
│  │  │ atomStateMap    │  │ mountedMap      │               ││
│  │  │ (WeakMap)       │  │ (WeakMap)       │               ││
│  │  └─────────────────┘  └─────────────────┘               ││
│  │  ┌─────────────────┐  ┌─────────────────┐               ││
│  │  │ readAtomState   │  │ writeAtomState  │               ││
│  │  │ mountAtom       │  │ invalidateDep...│               ││
│  │  └─────────────────┘  └─────────────────┘               ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

## 3. Atom 的定义 (`src/vanilla/atom.ts`)

```typescript
// Atom 接口
interface Atom<Value> {
  toString: () => string
  read: (get: Getter, options: { signal, setSelf }) => Value
  debugLabel?: string
}

// 可写 Atom
interface WritableAtom<Value, Args, Result> extends Atom<Value> {
  write: (get: Getter, set: Setter, ...args: Args) => Result
  onMount?: (setAtom) => OnUnmount | void
}

// atom() 工厂函数 - 函数重载支持多种创建方式
function atom(initialValue)           // 原始 atom
function atom(readFn)                 // 只读派生 atom
function atom(readFn, writeFn)        // 可写派生 atom
```

**关键点**：
- 每个 atom 有唯一的 `key`（通过全局 `keyCount` 生成）
- 原始 atom 的 `read` 默认返回 `get(this)`
- 原始 atom 的 `write` 默认调用 `set(this, value)`

## 4. Store 的核心状态 (`src/vanilla/internals.ts`)

```typescript
// Atom 状态 - 存储每个 atom 的运行时信息
type AtomState = {
  d: Map<Atom, number>    // 依赖图 + epoch number
  p: Set<Atom>            // 等待 Promise 的依赖 atoms
  n: number               // 当前 epoch（版本号）
  v?: Value               // 值
  e?: Error               // 错误
}

// 挂载状态 - 追踪订阅关系
type Mounted = {
  l: Set<() => void>      // 监听器集合
  d: Set<Atom>            // 依赖的 atoms
  t: Set<Atom>            // 被依赖的 atoms（订阅者）
  u?: () => void          // unmount 清理函数
}
```

## 5. 核心流程

### 5.1 读取 Atom 值 (`readAtomState`)

```
readAtomState(atom)
    │
    ├─► 检查缓存：mounted && 未 invalidated？
    │       └─► 是 → 返回缓存状态
    │
    ├─► 检查依赖是否变化
    │       └─► 否 → 返回缓存状态
    │
    └─► 重新计算
            │
            ├─► 清空依赖图
            ├─► 调用 atom.read(getter, options)
            │       └─► getter(otherAtom) 递归读取依赖
            ├─► 设置新值/错误
            ├─► 更新 epoch number
            └─► 返回新状态
```

### 5.2 写入 Atom (`writeAtomState`)

```
writeAtomState(atom, ...args)
    │
    ├─► 调用 atom.write(getter, setter, ...args)
    │       │
    │       └─► setter(targetAtom, value)
    │               ├─► targetAtom === atom?
    │               │       └─► 直接更新值
    │               │           └─► invalidateDependents(atom)
    │               └─► 递归调用 writeAtomState
    │
    └─► flushCallbacks() → 触发监听器重渲染
```

### 5.3 依赖追踪与失效传播

```
                 ┌──────────────┐
                 │  atomA (原始) │
                 └──────┬───────┘
                        │ d
              ┌─────────┴─────────┐
              ▼                   ▼
      ┌───────────────┐   ┌───────────────┐
      │ atomB (派生)   │   │ atomC (派生)   │
      │ d: [atomA]    │   │ d: [atomA]    │
      └───────┬───────┘   └───────┬───────┘
              │ t                 │ t
              └─────────┬─────────┘
                        ▼
                ┌───────────────┐
                │ atomD (派生)   │
                │ d: [atomB,C]  │
                └───────────────┘

当 atomA 变化时:
1. invalidateDependents(atomA) → 标记 atomB, atomC 失效
2. 递归标记 atomD 失效
3. flushCallbacks → 触发监听器
4. recomputeInvalidatedAtoms → 拓扑排序重新计算
```

## 6. React 集成

```typescript
// Provider.ts - 通过 Context 提供 Store
const StoreContext = createContext<Store | undefined>(undefined)

function Provider({ children, store }) {
  // 自动创建或使用传入的 store
  return <StoreContext.Provider value={store}>
    {children}
  </StoreContext.Provider>
}

// useAtomValue.ts - 订阅 atom 值变化
function useAtomValue(atom) {
  const store = useStore()

  // 使用 useReducer 触发重渲染
  const [[value], rerender] = useReducer(...)

  // 订阅变化
  useEffect(() => {
    return store.sub(atom, () => rerender())
  }, [store, atom])

  return value
}

// useSetAtom.ts - 获取 setter
function useSetAtom(atom) {
  const store = useStore()
  return useCallback((...args) => store.set(atom, ...args), [store, atom])
}
```

## 7. 关键设计特点

| 特点 | 实现方式 |
|------|----------|
| **自动依赖追踪** | 在 `readAtomState` 中，`getter` 函数自动记录依赖关系到 `atomState.d` |
| **惰性计算** | 只有被订阅的 atom 才会被挂载，未订阅的 atom 不会计算 |
| **细粒度更新** | 通过 epoch number 检测变化，避免不必要的重计算 |
| **拓扑排序更新** | `recomputeInvalidatedAtoms` 使用 DFS 拓扑排序确保正确的更新顺序 |
| **异步支持** | Promise 状态追踪 + AbortController + React 18 `use()` API |
| **内存安全** | 使用 WeakMap 存储 atom 状态，无引用时自动 GC |

## 8. 核心源码文件

| 文件 | 职责 |
|------|------|
| `src/vanilla/atom.ts` | Atom 类型定义和工厂函数 |
| `src/vanilla/store.ts` | Store 创建和默认 Store 管理 |
| `src/vanilla/internals.ts` | 核心状态管理逻辑（~1000行） |
| `src/react/Provider.ts` | React Context Provider |
| `src/react/useAtomValue.ts` | 订阅 atom 值变化 |
| `src/react/useSetAtom.ts` | 获取 atom 的 setter 函数 |
| `src/react/useAtom.ts` | 组合 useAtomValue + useSetAtom |

## 9. 与其他状态管理库的对比

| 特性 | Jotai | Redux | Recoil |
|------|-------|-------|--------|
| 状态单元 | Atom (任意粒度) | Single Store | Atom |
| 依赖管理 | 自动追踪 | 手动 selector | 手动 selector |
| 字符串 key | 不需要 | Action Type | 必需 |
| TypeScript | 原生支持 | 需要配置 | 需要配置 |
| Bundle 大小 | ~2KB | ~7KB | ~30KB |

## 10. 深入：Building Blocks 架构

Jotai 的 Store 采用了一个独特的"积木块"(Building Blocks)设计模式。所有核心功能都被拆分为独立的函数，存储在一个元组数组中：

```typescript
type BuildingBlocks = [
  // store state (索引 0-6)
  atomStateMap: AtomStateMap,           // WeakMap<Atom, AtomState>
  mountedMap: MountedMap,                // WeakMap<Atom, Mounted>
  invalidatedAtoms: InvalidatedAtoms,   // WeakMap<Atom, EpochNumber>
  changedAtoms: ChangedAtoms,            // Set<Atom>
  mountCallbacks: Callbacks,             // Set<() => void>
  unmountCallbacks: Callbacks,           // Set<() => void>
  storeHooks: StoreHooks,                // 生命周期钩子

  // atom interceptors (索引 7-10)
  atomRead: AtomRead,                    // 拦截 atom.read
  atomWrite: AtomWrite,                  // 拦截 atom.write
  atomOnInit: AtomOnInit,               // atom 初始化钩子
  atomOnMount: AtomOnMount,              // atom 挂载钩子

  // building-block functions (索引 11-20)
  ensureAtomState,                        // 确保创建 AtomState
  flushCallbacks,                         // 刷新回调队列
  recomputeInvalidatedAtoms,             // 重计算失效 atoms
  readAtomState,                          // 读取 atom 状态
  invalidateDependents,                   // 标记依赖失效
  writeAtomState,                         // 写入 atom 状态
  mountDependencies,                      // 挂载依赖
  mountAtom,                              // 挂载 atom
  unmountAtom,                            // 卸载 atom
  setAtomStateValueOrPromise,            // 设置值/Promise

  // store api (索引 21-23)
  storeGet,                               // store.get 实现
  storeSet,                               // store.set 实现
  storeSub,                               // store.sub 实现

  // enhance (索引 24)
  enhanceBuildingBlocks,                  // 增强函数（用于中间件）

  // abortable promise support (索引 25-27)
  abortHandlersMap,                       // Promise 中止处理器
  registerAbortHandler,                   // 注册中止处理器
  abortPromise,                            // 中止 Promise
]
```

**设计优势**：
1. **可扩展性**：通过 `enhanceBuildingBlocks` 可以替换任意函数实现中间件
2. **测试友好**：每个 Building Block 可独立测试
3. **Tree-shaking**：未使用的功能可被优化掉

## 11. 深入：异步处理机制

Jotai 对异步有完整的支持，包括 Promise 状态追踪和 AbortController。

### 11.1 Promise 状态追踪

```typescript
// AtomState 中的 p 字段追踪等待 Promise 的 atoms
type AtomState = {
  p: Set<Atom>  // 等待此 atom 的 Promise 解析的 atoms
  // ...
}

// 当 atom 的值是 Promise 时，追踪依赖关系
function addPendingPromiseToDependency(atom, promise, dependencyAtomState) {
  if (!dependencyAtomState.p.has(atom)) {
    dependencyAtomState.p.add(atom)
    const cleanup = () => dependencyAtomState.p.delete(atom)
    promise.then(cleanup, cleanup)  // Promise 完成后清理
  }
}
```

### 11.2 AbortController 支持

在 atom 的 read 函数中可以使用 AbortSignal：

```typescript
const dataAtom = atom(async (get, { signal }) => {
  const response = await fetch('/api/data', { signal })
  return response.json()
})
```

实现原理：

```typescript
// readAtomState 中创建 AbortController
let controller: AbortController | undefined
const options = {
  get signal() {
    if (!controller) {
      controller = new AbortController()
    }
    return controller.signal
  },
  // ...
}

// Promise 被中止时调用 abort
if (isPromiseLike(valueOrPromise)) {
  registerAbortHandler(store, valueOrPromise, () => controller?.abort())
}
```

### 11.3 Continuable Promise（React 集成）

用于 React Suspense 的特殊 Promise 处理：

```typescript
const createContinuablePromise = (store, promise, getValue) => {
  return new Promise((resolve, reject) => {
    let curr = promise

    const onAbort = () => {
      // 当 Promise 被中止时，尝试获取新值
      const nextValue = getValue()
      if (isPromiseLike(nextValue)) {
        // 继续追踪新的 Promise
        curr = nextValue
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

**作用**：当 Promise 值变化时（如重新请求），复用同一个 Promise 对象，避免 React 的"race condition"问题。

## 12. 深入：Epoch Number 变化检测机制

Jotai 使用 `epoch number` (版本号) 来检测变化，避免深度比较：

```typescript
type AtomState = {
  n: EpochNumber  // 每次值变化时递增
  d: Map<Atom, EpochNumber>  // 记录依赖的 epoch
  // ...
}
```

### 12.1 变化检测流程

```
┌─────────────────────────────────────────────────────────────┐
│                     readAtomState(atomA)                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  检查依赖: for (const [depAtom, recordedEpoch] of atomState.d) │
│    if (readAtomState(depAtom).n !== recordedEpoch) {        │
│      // 依赖已变化，需要重新计算                              │
│    }                                                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  setAtomStateValueOrPromise:                                │
│    if (!Object.is(prevValue, newValue)) {                  │
│      ++atomState.n  // 版本号递增                            │
│    }                                                         │
└─────────────────────────────────────────────────────────────┘
```

### 12.2 失效传播

```typescript
// invalidateDependents - 标记所有依赖者失效
const BUILDING_BLOCK_invalidateDependents = (store, atom) => {
  const stack = [atom]
  while (stack.length) {
    const a = stack.pop()!
    const aState = ensureAtomState(store, a)
    for (const dependent of getMountedOrPendingDependents(a, aState, mountedMap)) {
      const dState = ensureAtomState(store, dependent)
      // 记录失效时的 epoch，后续用于判断是否需要重计算
      if (invalidatedAtoms.get(dependent) !== dState.n) {
        invalidatedAtoms.set(dependent, dState.n)
        stack.push(dependent)  // 继续传播
      }
    }
  }
}
```

## 13. 深入：拓扑排序重计算

当多个 atoms 失效时，需要按依赖顺序重新计算：

```typescript
const BUILDING_BLOCK_recomputeInvalidatedAtoms = (store) => {
  // Step 1: 深度优先遍历，构建拓扑排序列表
  const topSortedReversed: [Atom, AtomState][] = []
  const visiting = new WeakSet<Atom>()
  const visited = new WeakSet<Atom>()
  const stack = Array.from(changedAtoms)

  while (stack.length) {
    const a = stack[stack.length - 1]

    if (visited.has(a)) {
      stack.pop()
      continue
    }

    if (visiting.has(a)) {
      // 所有依赖已处理，添加到结果
      if (invalidatedAtoms.get(a) === aState.n) {
        topSortedReversed.push([a, aState])
      }
      visited.add(a)
      stack.pop()
      continue
    }

    visiting.add(a)
    // 将未访问的依赖者压入栈
    for (const d of getMountedOrPendingDependents(a, aState, mountedMap)) {
      if (!visiting.has(d)) {
        stack.push(d)
      }
    }
  }

  // Step 2: 按拓扑顺序（反向）重计算
  for (let i = topSortedReversed.length - 1; i >= 0; --i) {
    const [a, aState] = topSortedReversed[i]
    let hasChangedDeps = false
    for (const dep of aState.d.keys()) {
      if (dep !== a && changedAtoms.has(dep)) {
        hasChangedDeps = true
        break
      }
    }
    if (hasChangedDeps) {
      readAtomState(store, a)
      mountDependencies(store, a)
    }
    invalidatedAtoms.delete(a)
  }
}
```

**算法特点**：
- 使用 DFS 实现拓扑排序
- 从变更的 atoms 开始，只遍历受影响的子图
- 支持环形依赖检测（通过 visiting/visited 标记）

## 14. 深入：React 集成细节

### 14.1 useAtomValue 完整流程

```typescript
function useAtomValue(atom, options) {
  const store = useStore(options)

  // 使用 useReducer 触发重渲染
  // 存储三元组：[value, store, atom]
  const [[valueFromReducer, storeFromReducer, atomFromReducer], rerender] =
    useReducer(
      (prev) => {
        const nextValue = store.get(atom)
        // 引用相等性检查，避免不必要重渲染
        if (
          Object.is(prev[0], nextValue) &&
          prev[1] === store &&
          prev[2] === atom
        ) {
          return prev
        }
        return [nextValue, store, atom]
      },
      undefined,
      () => [store.get(atom), store, atom]
    )

  // 处理 store/atom 变化
  let value = valueFromReducer
  if (storeFromReducer !== store || atomFromReducer !== atom) {
    rerender()  // 触发同步重渲染
    value = store.get(atom)
  }

  // 订阅变化
  useEffect(() => {
    const unsub = store.sub(atom, () => {
      // 处理 Promise 状态
      if (promiseStatus) {
        try {
          const value = store.get(atom)
          if (isPromiseLike(value)) {
            attachPromiseStatus(createContinuablePromise(store, value, () => store.get(atom)))
          }
        } catch {}
      }
      // delay 选项用于等待 Promise 解析
      if (typeof delay === 'number') {
        setTimeout(rerender, delay)
        return
      }
      rerender()
    })
    rerender()  // 初始渲染
    return unsub
  }, [store, atom, delay, promiseStatus])

  // 处理 Promise 值（Suspense）
  if (isPromiseLike(value)) {
    const promise = createContinuablePromise(store, value, () => store.get(atom))
    if (promiseStatus) {
      attachPromiseStatus(promise)
    }
    return use(promise)  // React 18+ 的 use() API
  }

  return value
}
```

### 14.2 React 18 Suspense 支持

Jotai 使用 React 18 的 `use()` API 支持 Suspense：

```typescript
// polyfill for older React
const use = React.use || ((promise) => {
  if (promise.status === 'pending') {
    throw promise  // Suspense: 抛出 Promise
  } else if (promise.status === 'fulfilled') {
    return promise.value
  } else if (promise.status === 'rejected') {
    throw promise.reason
  } else {
    attachPromiseStatus(promise)
    throw promise
  }
})

// 给 Promise 添加 status 属性
const attachPromiseStatus = (promise) => {
  if (!promise.status) {
    promise.status = 'pending'
    promise.then(
      (v) => { promise.status = 'fulfilled'; promise.value = v },
      (e) => { promise.status = 'rejected'; promise.reason = e }
    )
  }
}
```

## 15. 深入：工具函数实现

### 15.1 splitAtom - 数组拆分

将数组 atom 拆分为独立的元素 atoms：

```typescript
const numbersAtom = atom([1, 2, 3])
const splitAtoms = splitAtom(numbersAtom)

// splitAtoms 是一个派生 atom，值为 [atom1, atom2, atom3]
// 每个 atom1 可以独立读写
```

**实现要点**：
- 使用 WeakMap 缓存映射关系
- 通过 keyExtractor 支持自定义键
- 写入时自动更新原数组

```typescript
const read = (get) => {
  const arr = get(arrAtom)
  const mapping = getMapping(arr, prev?.arr)
  // 返回每个元素的 atom 列表
  return mapping.atomList
}

const write = (get, set, action) => {
  switch (action.type) {
    case 'remove':
      // 从数组中移除元素
      const index = get(splittedAtom).indexOf(action.atom)
      set(arrAtom, [...arr.slice(0, index), ...arr.slice(index + 1)])
      break
    case 'insert':
      // 插入元素
      // ...
  }
}
```

### 15.2 atomFamily - 参数化 atom（已弃用）

创建基于参数的 atom 工厂：

```typescript
const todoFamily = atomFamily((id: string) => atom(`todo-${id}`))

// 相同 id 返回相同的 atom
todoFamily('a') === todoFamily('a')  // true
```

**实现原理**：
- 使用 Map 缓存已创建的 atoms
- 支持自定义比较函数
- 支持 TTL 清理（setShouldRemove）

## 16. 性能优化策略

### 16.1 WeakMap 内存管理

```typescript
// AtomState 和 Mounted 都用 WeakMap 存储
// 当 atom 对象被 GC 时，相关状态自动清理
type AtomStateMap = WeakMap<Atom, AtomState>
type MountedMap = WeakMap<Atom, Mounted>
```

### 16.2 批量更新

```typescript
const BUILDING_BLOCK_flushCallbacks = (store) => {
  const changedAtoms = buildingBlocks[3]
  const mountCallbacks = buildingBlocks[4]
  const unmountCallbacks = buildingBlocks[5]

  do {
    const callbacks = new Set()
    // 收集所有变更 atom 的监听器
    changedAtoms.forEach(atom => mountedMap.get(atom)?.l.forEach(cb => callbacks.add(cb)))
    changedAtoms.clear()

    // 执行所有回调
    callbacks.forEach(fn => fn())

    // 如果有新的变更，继续循环
  } while (changedAtoms.size || unmountCallbacks.size || mountCallbacks.size)
}
```

### 16.3 惰性计算

只有被订阅的 atom 才会被挂载和计算：

```typescript
const BUILDING_BLOCK_mountAtom = (store, atom) => {
  let mounted = mountedMap.get(atom)
  if (!mounted) {
    // 首次挂载时才读取状态
    readAtomState(store, atom)
    // 递归挂载依赖
    for (const dep of atomState.d.keys()) {
      mountAtom(store, dep)
    }
    mounted = { l: new Set(), d: new Set(), t: new Set() }
    mountedMap.set(atom, mounted)
  }
  return mounted
}
```

## 17. 总结

Jotai 通过以下机制实现了轻量、高效的状态管理：

1. **原子化状态**：每个 atom 是独立的状态单元，可自由组合
2. **自动依赖追踪**：读取时自动建立依赖关系，无需手动声明
3. **惰性求值**：只有被订阅时才计算，避免不必要的计算
4. **细粒度响应式更新**：通过 epoch number 和失效传播机制实现精确更新
5. **拓扑排序重计算**：确保依赖链正确更新顺序
6. **异步原生支持**：Promise 状态追踪 + AbortController + Suspense
7. **内存安全**：WeakMap 自动 GC，无内存泄漏风险
8. **框架无关核心**：Vanilla Store 可独立于 React 使用
9. **可扩展架构**：Building Blocks 模式支持中间件和调试工具