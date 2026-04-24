# Jotai 实现原理详解

## 1. 整体架构

Jotai 采用分层架构设计：

```
┌─────────────────────────────────────────────┐
│           React Layer (react/)              │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐ │
│  │ useAtom  │ │useAtomValue│ │ useSetAtom │ │
│  └──────────┘ └──────────┘ └──────────────┘ │
├─────────────────────────────────────────────┤
│         Vanilla Layer (vanilla/)            │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐ │
│  │  atom()  │ │  store   │ │  internals   │ │
│  └──────────┘ └──────────┘ └──────────────┘ │
└─────────────────────────────────────────────┘
```

核心设计原则：
- **Vanilla 核心**：与框架无关的状态管理逻辑
- **React 绑定**：通过 Hooks 与 React 集成
- **原子化状态**：所有状态以 Atom 为基本单位

---

## 2. Atom 系统

### 2.1 Atom 类型定义

```typescript
// 基础 Atom 接口
export interface Atom<Value> {
  toString: () => string
  read: Read<Value>
  debugLabel?: string
  debugPrivate?: boolean
}

// 可写 Atom 接口
export interface WritableAtom<Value, Args extends unknown[], Result> extends Atom<Value> {
  read: Read<Value, SetAtom<Args, Result>>
  write: Write<Args, Result>
  onMount?: OnMount<Args, Result>
}

// 原始值 Atom（通过 atom(initialValue) 创建）
export interface PrimitiveAtom<Value> extends WritableAtom<Value, [SetStateAction<Value>], void> {}
```

### 2.2 Atom 创建函数

```typescript
// 重载 1: 创建派生 Atom（只读）
export function atom<Value>(read: Read<Value>): Atom<Value>

// 重载 2: 创建可写派生 Atom
export function atom<Value, Args extends unknown[], Result>(
  read: Read<Value, SetAtom<Args, Result>>,
  write: Write<Args, Result>,
): WritableAtom<Value, Args, Result>

// 重载 3: 创建原始值 Atom
export function atom<Value>(initialValue: Value): PrimitiveAtom<Value> & WithInitialValue<Value>
```

### 2.3 Atom 内部状态 (AtomState)

```typescript
type AtomState<Value = AnyValue> = {
  // 依赖图：atom -> epoch number
  // 记录该 atom 依赖的其他 atoms 及其版本号
  readonly d: Map<AnyAtom, EpochNumber>

  // 挂起的 Promise 集合
  // 当 read 函数返回 Promise 时使用
  readonly p: Set<AnyAtom>

  // 当前 epoch number（版本号）
  // 每次值改变时递增
  n: EpochNumber

  // 当前值（可能不存在）
  v?: Value

  // 错误（如果 read 抛出异常）
  e?: AnyError
}
```

---

## 3. 依赖追踪与状态更新

### 3.1 依赖追踪机制

依赖追踪发生在 `readAtomState` 函数中，核心是一个包装过的 getter 函数：

```typescript
const BUILDING_BLOCK_readAtomState: ReadAtomState = (store, atom) => {
  const atomState = ensureAtomState(store, atom)

  // 检查是否可以跳过重新计算（缓存命中）
  if (isAtomStateInitialized(atomState)) {
    // 检查依赖是否发生变化
    let hasChangedDeps = false
    for (const [a, n] of atomState.d) {
      if (readAtomState(store, a).n !== n) {
        hasChangedDeps = true
        break
      }
    }
    if (!hasChangedDeps) return atomState  // 依赖未变，使用缓存
  }

  // 清空旧依赖，准备重新计算
  atomState.d.clear()

  // 创建带依赖追踪的 getter 函数
  function getter<V>(a: Atom<V>) {
    if (isSelfAtom(atom, a)) {
      const aState = readAtomState(store, a)
      addDependency(atom, a, aState)
      return returnAtomValue(aState)
    }
    // 递归读取其他 atom
    const aState = readAtomState(store, a)
    try {
      return returnAtomValue(aState)
    } finally {
      // 关键：记录依赖关系！
      atomState.d.set(a, aState.n)
      // 同时建立反向依赖（被依赖关系）
      if (hasPromiseAtomValue(aState)) {
        atomState.p.add(a)
      }
    }
  }

  // 执行 atom 的 read 函数，传入追踪过的 getter
  const valueOrPromise = atom.read(getter, options)

  // 处理返回值...
}
```

### 3.2 依赖关系可视化

```
Atom A (派生 Atom)
  ├── 依赖 Atom B ── getter(B) ── 记录到 AtomState(A).d
  ├── 依赖 Atom C ── getter(C) ── 记录到 AtomState(A).d
  └── 依赖 Atom D ── getter(D) ── 记录到 AtomState(A).d

AtomState(A).d = { B -> epoch(B), C -> epoch(C), D -> epoch(D) }
```

### 3.3 状态更新与级联更新

当调用 `setAtom` 或 `atom.write` 时，触发更新流程：

```typescript
const BUILDING_BLOCK_writeAtomState: WriteAtomState = (store, atom, ...args) => {
  const buildingBlocks = getInternalBuildingBlocks(store)
  const [, , readAtomState] = buildingBlocks

  // 创建 setter 函数
  const setter: Setter = <V, As extends unknown[], R>(
    a: WritableAtom<V, As, R>,
    ...args: As
  ) => {
    const aState = ensureAtomState(store, a)
    try {
      if (a === (atom as AnyAtom)) {
        // ===== 更新当前 atom =====
        const prevEpochNumber = aState.n
        const v = args[0] as V

        // 1. 设置新值
        setAtomStateValueOrPromise(store, a, v)

        // 2. 重新挂载依赖
        mountDependencies(store, a)

        // 3. 如果值确实改变了（epoch 变化）
        if (prevEpochNumber !== aState.n) {
          changedAtoms.add(a)                    // 标记为已改变
          invalidateDependents(store, a)         // 使依赖者失效！
          storeHooks.c?.(a)                      // 调用 onChange 钩子
        }
        return undefined as R
      } else {
        // ===== 递归更新其他 atom =====
        return writeAtomState(store, a, ...args)
      }
    } finally {
      // 异步时重新计算和刷新
      if (!isSync) {
        recomputeInvalidatedAtoms(store)
        flushCallbacks(store)
      }
    }
  }

  // 执行 atom 的 write 函数
  return atom.write(getter, setter, ...args)
}
```

### 3.4 使依赖者失效 (invalidateDependents)

```typescript
const BUILDING_BLOCK_invalidateDependents: InvalidateDependents = (store, atom) => {
  const [, , readAtomState] = buildingBlocks
  const mounted = mountedMap.get(atom)
  if (mounted) {
    // mounted.t 包含所有依赖此 atom 的 atoms（被依赖者列表）
    for (const dependent of mounted.t) {
      if (!invalidatedAtoms.has(dependent)) {
        invalidatedAtoms.set(dependent, atomState.n)  // 标记为失效
        invalidateDependents(store, dependent)           // 递归使下游失效
      }
    }
  }
}
```

**失效传播示例**：
```
Atom A 被更新
  └── invalidateDependents(A)
        └── 找到所有依赖 A 的 atoms: [B, C]
              ├── invalidatedAtoms.set(B, epoch)
              │   └── invalidateDependents(B)
              │       └── 找到依赖 B 的 atoms: [D]
              │           └── invalidatedAtoms.set(D, epoch)
              └── invalidatedAtoms.set(C, epoch)
```

### 3.5 重新计算失效的 Atoms

```typescript
const BUILDING_BLOCK_recomputeInvalidatedAtoms: RecomputeInvalidatedAtoms = (store) => {
  const [, , readAtomState, , , , , , , , , , , , , mountDependencies] = buildingBlocks

  // 用于拓扑排序的数组
  const topSortedReversed: [atom: AnyAtom, atomState: AtomState][] = []

  // 标记已访问的 atom
  const markedAtoms = new Set<AnyAtom>()
  const visiting = new Set<AnyAtom>()  // 用于检测循环依赖

  // 深度优先搜索遍历依赖图
  const stack: AnyAtom[] = Array.from(changedAtoms)
  while (stack.length) {
    const a = stack[stack.length - 1]!
    if (markedAtoms.has(a)) {
      stack.pop()
      continue
    }

    let aState = readAtomState(store, a)

    if (!visiting.has(a)) {
      visiting.add(a)
      // 将依赖压入栈（后序遍历）
      for (const [dep] of aState.d) {
        if (!markedAtoms.has(dep) && !changedAtoms.has(dep)) {
          stack.push(dep)
        }
      }
    }

    if (stack[stack.length - 1] === a) {
      visiting.delete(a)
      markedAtoms.add(a)
      topSortedReversed.push([a, aState])
      stack.pop()
    }
  }

  // 按拓扑排序的逆序重新计算所有受影响的 atom
  for (let i = topSortedReversed.length - 1; i >= 0; --i) {
    const [a, aState] = topSortedReversed[i]!
    let hasChangedDeps = false
    for (const dep of aState.d.keys()) {
      if (dep !== a && changedAtoms.has(dep)) {
        hasChangedDeps = true
        break
      }
    }
    if (hasChangedDeps) {
      readAtomState(store, a)  // 重新计算
      mountDependencies(store, a)
    }
  }
}
```

---

## 4. 订阅系统

### 4.1 Store 订阅机制

```typescript
const BUILDING_BLOCK_storeSub: StoreSub = (store, atom, listener) => {
  const buildingBlocks = getInternalBuildingBlocks(store)
  const flushCallbacks = buildingBlocks[12]
  const mountAtom = buildingBlocks[18]
  const unmountAtom = buildingBlocks[19]

  // 1. 挂载 atom（创建 Mounted 状态）
  const mounted = mountAtom(store, atom)

  // 2. 获取监听器集合，添加新监听器
  const listeners = mounted.l
  listeners.add(listener)

  // 3. 刷新回调队列
  flushCallbacks(store)

  // 4. 返回取消订阅函数
  return () => {
    listeners.delete(listener)  // 移除监听器
    unmountAtom(store, atom)     // 尝试卸载 atom
    flushCallbacks(store)
  }
}
```

### 4.2 挂载状态 (Mounted)

```typescript
type Mounted = {
  // l: listeners - 订阅者集合
  readonly l: Set<() => void>

  // d: dependencies - 已挂载的依赖 atoms
  readonly d: Set<AnyAtom>

  // t: dependents - 依赖于本 atom 的 atoms（被依赖者）
  readonly t: Set<AnyAtom>

  // u: unmount - 卸载时的清理函数（可选）
  u?: () => void
}
```

**可视化关系**：
```
Atom A 的 Mounted 状态:
  l: { listener1, listener2, ... }     ← 订阅者（通常是 React 组件）
  d: { Atom B, Atom C }                ← A 依赖的 atoms
  t: { Atom D, Atom E }                  ← 依赖 A 的 atoms
  u: () => { ... }                      ← onMount 返回的清理函数
```

### 4.3 Mount 和 Unmount 流程

```typescript
const BUILDING_BLOCK_mountAtom: MountAtom = (store, atom) => {
  const buildingBlocks = getInternalBuildingBlocks(store)
  const [, atomStateMap, readAtomState, , , , , , , , , , , , , mountDependencies] = buildingBlocks

  const atomState = ensureAtomState(store, atom)
  let mounted = mountedMap.get(atom)

  if (!mounted) {
    // 1. 首次挂载，创建 Mounted 对象
    mounted = {
      l: new Set(),           // 监听器集合
      d: new Set(),           // 依赖集合
      t: new Set(),           // 被依赖者集合
    }
    mountedMap.set(atom, mounted)

    // 2. 确保 atom 已初始化（触发 read）
    readAtomState(store, atom)

    // 3. 挂载所有依赖
    mountDependencies(store, atom)

    // 4. 调用 onMount 生命周期（如果定义）
    const { onMount } = atom as AnyWritableAtom
    if (onMount) {
      const { c: CONTINUATION_PROMISE_WHEN_USED_IN_MOUNT } = atomState
      const setAtom = (...args: unknown[]) => {
        if (CONTINUATION_PROMISE_WHEN_USED_IN_MOUNT) {
          return CONTINUATION_PROMISE_WHEN_USED_IN_MOUNT.then(() =>
            store.set(atom as AnyWritableAtom, ...args),
          )
        }
        return store.set(atom as AnyWritableAtom, ...args)
      }
      const cleanup = onMount(...args)
      if (cleanup) {
        mounted.u = cleanup  // 保存清理函数
      }
    }
  }

  return mounted
}
```

**Unmount 流程**（简化）：
```typescript
const unmountAtom = (store, atom) => {
  // 1. 检查是否还有其他订阅者
  if (!mounted.l.size && !atomState.p.size) {
    // 2. 调用清理函数
    if (mounted.u) mounted.u()

    // 3. 卸载所有依赖
    for (const dep of mounted.d) {
      // 从依赖的 t 集合中移除当前 atom
      const depMounted = mountedMap.get(dep)
      depMounted?.t.delete(atom)
    }

    // 4. 删除 mounted 状态
    mountedMap.delete(atom)

    // 5. 清理 atomState（可选）
    if (typeof atom.unmount === 'function') {
      atom.unmount()
    }
  }
}
```

---

## 5. React 集成

### 5.1 Store Provider

```typescript
// React Context 传递 Store
const StoreContext = createContext<Store | undefined>(undefined)

export const Provider = ({ store, children }) => {
  const storeRef = useRef(store || createStore())
  return (
    <StoreContext.Provider value={storeRef.current}>
      {children}
    </StoreContext.Provider>
  )
}

// Hook 获取 Store
export const useStore = (options?: { store?: Store }) => {
  const store = useContext(StoreContext)
  return options?.store || store || getDefaultStore()
}
```

### 5.2 useAtomValue Hook

```typescript
export function useAtomValue<Value>(atom: Atom<Value>, options?: Options) {
  const store = useStore(options)

  // 使用 useReducer 强制重新渲染
  const [[valueFromReducer, storeFromReducer, atomFromReducer], rerender] =
    useReducer<
      Reducer<
        readonly [Value, Store, typeof atom],
        undefined
      >,
      undefined
    >(
      (prev, _action?) => {
        const nextValue = store.get(atom)
        if (
          Object.is(prev[0], nextValue) &&
          prev[1] === store &&
          prev[2] === atom
        ) {
          return prev // 值相同，不重新渲染
        }
        return [nextValue, store, atom]
      },
      undefined,
      () => [store.get(atom), store, atom],
    )

  // 订阅 atom 变化
  useEffect(() => {
    const unsub = store.sub(atom, () => {
      if (typeof delay === 'number') {
        setTimeout(rerender, delay)
        return
      }
      rerender() // atom 变化时触发重新渲染
    })
    rerender()
    return unsub
  }, [store, atom, delay])

  // 处理 Promise（Suspense 支持）
  let value = valueFromReducer
  if (
    storeFromReducer !== store ||
    atomFromReducer !== atom
  ) {
    rerender()
    value = store.get(atom)
  }
  if (isPromiseLike(value)) {
    const promise = createContinuablePromise(value, () => store.get(atom))
    return use(promise)
  }
  return value as Awaited<Value>
}
```

### 5.3 useSetAtom Hook

```typescript
export function useSetAtom<Value, Args extends unknown[], Result>(
  atom: WritableAtom<Value, Args, Result>,
  options?: Options,
): SetAtom<Args, Result> {
  const store = useStore(options)
  return useMemo(() => {
    const setAtom = (...args: Args) => {
      return (store as Store).set(atom, ...args)
    }
    return setAtom
  }, [store, atom])
}
```

### 5.4 useAtom Hook（组合 useAtomValue 和 useSetAtom）

```typescript
export function useAtom<Value, Args extends unknown[], Result>(
  atom: WritableAtom<Value, Args, Result>,
  options?: Options,
): [Awaited<Value>, SetAtom<Args, Result>]

export function useAtom<Value>(
  atom: Atom<Value>,
  options?: Options,
): [Awaited<Value>, never]

export function useAtom<Value, Args extends unknown[], Result>(
  atom: WritableAtom<Value, Args, Result> | Atom<Value>,
  options?: Options,
) {
  return [
    useAtomValue(atom, options),
    // 类型断言是安全的，因为只读 atom 的 SetAtom 返回 never
    useSetAtom(atom as WritableAtom<Value, Args, Result>, options),
  ]
}
```

---

## 6. 核心数据结构与算法总结

### 6.1 核心数据结构

| 结构 | 类型 | 作用 |
|------|------|------|
| `Atom` | Interface | 用户可见的 atom 定义（read/write） |
| `AtomState` | Object | atom 的内部状态（值、依赖、epoch） |
| `Mounted` | Object | 已挂载 atom 的运行时信息 |
| `atomStateMap` | WeakMap | Store 中所有 atom 的状态存储 |
| `mountedMap` | WeakMap | 已挂载 atom 的信息存储 |
| `invalidatedAtoms` | WeakMap | 标记失效的 atoms |

### 6.2 核心算法流程

**读取 Atom（带缓存和依赖追踪）**:
```
readAtomState(atom)
  ↓
检查 AtomState 是否已初始化且有有效缓存
  ↓
是 → 检查依赖是否变化
  ↓     ↓
  ↓   无变化 → 返回缓存值
  ↓     ↓
  ↓   有变化 → 继续计算
  ↓
否 → 继续计算
  ↓
清空旧依赖 (atomState.d.clear())
  ↓
执行 atom.read(getter)
  ↓
getter 访问其他 atom 时:
  - 递归调用 readAtomState(dep)
  - atomState.d.set(dep, depState.n) ← 记录依赖
  ↓
保存计算结果到 atomState.v
  ↓
返回 atomState
```

**更新 Atom（带级联更新）**:
```
writeAtomState(atom, value)
  ↓
执行 atom.write(getter, setter, args)
  ↓
setter 被调用:
  - 更新 atomState.v
  - atomState.n++ (递增 epoch)
  - changedAtoms.add(atom)
  - invalidateDependents(atom) ← 使依赖者失效
  - storeHooks.c?.(atom) ← 通知变更
  ↓
invalidateDependents(atom)
  ↓
遍历 atom 的 mounted.t（被依赖者列表）
  ↓
对每个 dependent:
  - invalidatedAtoms.set(dependent, epoch)
  - invalidateDependents(dependent) ← 递归
  ↓
recomputeInvalidatedAtoms()
  ↓
拓扑排序（DFS）所有受影响的 atoms
  ↓
按逆拓扑顺序重新计算每个 atom
  ↓
通知订阅者（调用 mounted.l 中的监听器）
```

---

## 7. 关键设计亮点

### 7.1 Epoch Number 版本控制

使用递增的整数 (`epoch number`) 而非深比较来判断值是否变化：

```typescript
// 设置新值时
const prevEpochNumber = aState.n
setAtomStateValueOrPromise(store, a, v)
if (prevEpochNumber !== aState.n) {
  // 值确实改变了
  changedAtoms.add(a)
}
```

**优点**：
- O(1) 时间复杂度判断变化
- 避免深比较的性能开销
- 天然支持不可变更新模式

### 7.2 惰性计算 (Lazy Evaluation)

Atom 的值只在被读取时才计算：

```typescript
// 创建 atom 时不会立即计算
const derivedAtom = atom((get) => get(baseAtom) * 2)

// 只有在 useAtomValue(derivedAtom) 或 get(derivedAtom) 时才计算
```

**优点**：
- 避免不必要的计算
- 支持动态依赖（条件依赖）
- 优化启动性能

### 7.3 自动依赖追踪

无需手动声明依赖，通过 getter 函数自动追踪：

```typescript
const sumAtom = atom((get) => {
  // 以下对 get() 的调用被自动记录为依赖
  const a = get(atomA)  // → sumAtomState.d.set(atomA, epoch)
  const b = get(atomB)  // → sumAtomState.d.set(atomB, epoch)
  return a + b
})
```

**优点**：
- 无需手动维护依赖数组
- 动态依赖自动处理
- 避免遗漏依赖导致的 bug

### 7.4 精确的级联更新

使用拓扑排序确保正确的更新顺序：

```typescript
// 假设依赖关系：A ← B ← C，A ← D
// 当 A 更新时：

// 1. invalidateDependents(A) 标记 B 和 D 为失效
// 2. invalidateDependents(B) 递归标记 C 为失效
// 3. recomputeInvalidatedAtoms() 拓扑排序

// 拓扑排序结果（逆序）：[A, B, D, C] 或 [A, D, B, C]
// 重新计算顺序（正序）：C → B → D → A（或 C → D → B → A）

// 确保 C 在 B 之后计算，因为 C 依赖 B
```

**优点**：
- 正确的计算顺序
- 避免重复计算
- 支持复杂的依赖图（包括循环依赖检测）

### 7.5 高效的订阅管理

使用 WeakMap 和 Set 实现内存安全和高效查找：

```typescript
// 存储结构
const atomStateMap = new WeakMap<AnyAtom, AtomState>()  // atom → 状态
const mountedMap = new WeakMap<AnyAtom, Mounted>()      // atom → 挂载信息
const invalidatedAtoms = new WeakMap<AnyAtom, EpochNumber>() // 失效标记

// Mounted.l 是 Set，提供 O(1) 的增删查
mounted.l.add(listener)      // O(1)
mounted.l.delete(listener)   // O(1)
mounted.l.forEach(notify)    // O(n)
```

**优点**：
- WeakMap 自动垃圾回收
- Set 提供高效操作
- 订阅者数量不影响读取性能

---

## 8. 与 React 的集成机制

### 8.1 Store 在 React 树中的传递

```typescript
// Context 定义
const StoreContext = createContext<Store | undefined>(undefined)

// Provider 组件
export const Provider = ({
  store,
  children,
}: {
  store?: Store
  children: ReactNode
}) => {
  const storeRef = useRef(store)
  if (!storeRef.current) {
    storeRef.current = createStore()
  }
  return (
    <StoreContext.Provider value={storeRef.current}>
      {children}
    </StoreContext.Provider>
  )
}

// Hook 获取 Store
export const useStore = (options?: { store?: Store }): Store => {
  const store = useContext(StoreContext)
  return options?.store || store || getDefaultStore()
}
```

### 8.2 触发重新渲染的机制

使用 `useReducer` 强制组件重新渲染：

```typescript
const [, rerender] = useReducer<
  Reducer<readonly [Value, Store, typeof atom], undefined>
>(
  (prev) => {
    const nextValue = store.get(atom)
    // 使用 Object.is 进行精确比较
    if (
      Object.is(prev[0], nextValue) &&
      prev[1] === store &&
      prev[2] === atom
    ) {
      return prev // 值相同，不触发重新渲染
    }
    return [nextValue, store, atom]
  },
  undefined,
  () => [store.get(atom), store, atom],
)
```

**为什么使用 useReducer 而不是 useState？**
- `useReducer` 可以比较前一个值和新值，决定是否更新
- 避免不必要的重新渲染
- 支持批处理和优化

### 8.3 订阅与取消订阅

```typescript
useEffect(() => {
  // 订阅 atom 变化
  const unsub = store.sub(atom, () => {
    if (typeof delay === 'number') {
      setTimeout(rerender, delay) // 延迟重新渲染
      return
    }
    rerender() // 立即重新渲染
  })

  // 立即执行一次，确保状态同步
  rerender()

  // 返回取消订阅函数
  return unsub
}, [store, atom, delay])
```

### 8.4 Suspense 支持

处理 Promise 类型的 atom 值：

```typescript
if (isPromiseLike(value)) {
  // 创建可继续的 Promise
  const promise = createContinuablePromise(value, () => store.get(atom))

  // 使用 React 18+ 的 use() Hook
  // 这将触发 Suspense fallback
  return use(promise)
}
```

---

## 9. 性能优化策略

### 9.1 惰性求值

Atom 的值只在被读取时才计算：

```typescript
const expensiveAtom = atom((get) => {
  // 复杂的计算...
  return result
})

// 创建 atom 时不执行
// 只有 useAtomValue(expensiveAtom) 或 get(expensiveAtom) 时才执行
```

### 9.2 精确更新

通过 epoch number 精确判断是否需要更新：

```typescript
const prevEpochNumber = aState.n
setAtomStateValueOrPromise(store, a, v)
if (prevEpochNumber !== aState.n) {
  // 只有真正改变时才触发更新
  changedAtoms.add(a)
  invalidateDependents(store, a)
}
```

### 9.3 细粒度订阅

组件只订阅需要的 atom，而非整个 store：

```typescript
function ComponentA() {
  const [value] = useAtom(atomA)  // 只订阅 atomA
  return <div>{value}</div>
}

function ComponentB() {
  const [value] = useAtom(atomB)  // 只订阅 atomB
  return <div>{value}</div>
}

// atomA 变化时只有 ComponentA 重新渲染
// atomB 变化时只有 ComponentB 重新渲染
```

### 9.4 依赖缓存

通过缓存依赖关系避免重复计算：

```typescript
// 检查依赖是否变化
let hasChangedDeps = false
for (const [a, n] of atomState.d) {
  if (readAtomState(store, a).n !== n) {
    hasChangedDeps = true
    break
  }
}
if (!hasChangedDeps) {
  return atomState  // 依赖未变，直接返回缓存
}
```

### 9.5 拓扑排序更新

通过拓扑排序确保正确的更新顺序：

```typescript
// 构建依赖图并拓扑排序
const topSortedReversed: [AnyAtom, AtomState][] = []
// ... DFS 构建拓扑序

// 按逆拓扑顺序重新计算
for (let i = topSortedReversed.length - 1; i >= 0; --i) {
  const [a, aState] = topSortedReversed[i]!
  // 重新计算 atom
}
```

---

## 10. 总结

Jotai 是一个设计精良的状态管理库，其核心特点包括：

### 10.1 原子化设计

- 状态以 Atom 为基本单位，每个 Atom 独立管理
- 支持派生 Atom（基于其他 Atom 计算）
- 细粒度的状态管理，避免不必要的重新渲染

### 10.2 自动依赖追踪

- 无需手动声明依赖
- 通过 getter 函数自动追踪依赖关系
- 支持动态依赖（条件依赖）

### 10.3 高效的更新机制

- 使用 epoch number 精确判断变化
- 拓扑排序确保正确的更新顺序
- 惰性求值避免不必要的计算

### 10.4 灵活的订阅系统

- 基于 Store 的订阅模型
- 支持多 Store（通过 Provider）
- 自动挂载/卸载管理

### 10.5 与 React 无缝集成

- 提供直观的 Hooks API（useAtom, useAtomValue, useSetAtom）
- 支持 React 18 Concurrent Features（use Hook, Suspense）
- 支持延迟重新渲染

### 10.6 性能优化

- 细粒度订阅避免不必要的渲染
- 精确更新机制
- 依赖缓存和惰性求值

Jotai 的设计哲学是**简单、灵活、高效**，它借鉴了 Recoil 的原子化概念，但提供了更轻量级、更灵活的实现。通过理解其内部机制，开发者可以更好地利用 Jotai 构建高性能的 React 应用。
