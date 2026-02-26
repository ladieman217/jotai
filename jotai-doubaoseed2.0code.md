# Jotai 实现原理深度解析

本文从底层算法、数据结构设计、边缘案例处理等角度深入剖析 Jotai 的实现原理。

---

## 目录

1. [核心数据结构设计](#1-核心数据结构设计)
2. [版本号机制与依赖追踪](#2-版本号机制与依赖追踪)
3. [读取原子的完整流程](#3-读取原子的完整流程)
4. [写入与失效传播](#4-写入与失效传播)
5. [拓扑排序重计算算法](#5-拓扑排序重计算算法)
6. [挂载生命周期管理](#6-挂载生命周期管理)
7. [React 集成的微妙之处](#7-react-集成的微妙之处)
8. [异步 Promise 处理](#8-异步-promise-处理)
9. [内存管理与 GC 友好设计](#9-内存管理与-gc-友好设计)

---

## 1. 核心数据结构设计

### 1.1 BuildingBlocks 元组架构

在 `src/vanilla/internals.ts:158-192` 中，Jotai 使用一个长度为 28 的元组来组织所有内部状态和函数：

```typescript
type BuildingBlocks = [
  // [0-6] 存储状态
  atomStateMap: AtomStateMap,           // 0: 所有原子状态
  mountedMap: MountedMap,               // 1: 已挂载原子状态
  invalidatedAtoms: InvalidatedAtoms,   // 2: 失效原子映射
  changedAtoms: ChangedAtoms,           // 3: 变化原子集合
  mountCallbacks: Callbacks,            // 4: 挂载回调
  unmountCallbacks: Callbacks,          // 5: 卸载回调
  storeHooks: StoreHooks,               // 6: 存储钩子

  // [7-10] 原子拦截器
  atomRead: AtomRead,                    // 7: 原子读取拦截器
  atomWrite: AtomWrite,                  // 8: 原子写入拦截器
  atomOnInit: AtomOnInit,                // 9: 原子初始化钩子
  atomOnMount: AtomOnMount,              // 10: 原子挂载钩子

  // [11-20] 核心构建块函数
  ensureAtomState: EnsureAtomState,      // 11: 确保原子状态存在
  flushCallbacks: FlushCallbacks,        // 12: 刷新回调
  recomputeInvalidatedAtoms: ...,        // 13: 重计算失效原子
  readAtomState: ReadAtomState,          // 14: 读取原子状态
  invalidateDependents: ...,              // 15: 失效依赖者
  writeAtomState: WriteAtomState,        // 16: 写入原子状态
  mountDependencies: MountDependencies,  // 17: 挂载依赖
  mountAtom: MountAtom,                  // 18: 挂载原子
  unmountAtom: UnmountAtom,              // 19: 卸载原子
  setAtomStateValueOrPromise: ...,       // 20: 设置原子值

  // [21-27] Store API 和其他
  storeGet: StoreGet,                    // 21: store.get 实现
  storeSet: StoreSet,                    // 22: store.set 实现
  storeSub: StoreSub,                    // 23: store.sub 实现
  enhanceBuildingBlocks: ...,             // 24: 增强构建块
  abortHandlersMap: AbortHandlersMap,    // 25: 中止处理器映射
  registerAbortHandler: ...,              // 26: 注册中止处理器
  abortPromise: AbortPromise,            // 27: 中止 Promise
]
```

**设计意图**：这种元组设计允许 Jotai 通过数组索引快速访问各个组件，同时支持通过 `buildArgs` 覆盖任意组件，实现高度可扩展性。

### 1.2 AtomState 的详细结构

```typescript
type AtomState<Value = AnyValue> = {
  // 依赖映射：Map<依赖原子, 依赖时的版本号>
  readonly d: Map<AnyAtom, EpochNumber>

  // 有 pending promise 的依赖者集合
  // 注意：这可能造成内存泄漏，因为这些原子即使没有被订阅也会被保留
  readonly p: Set<AnyAtom>

  // 版本号（epoch number）：每次值变化时递增
  // 这是 Jotai 依赖追踪的核心机制
  n: EpochNumber

  // 原子值：可能是 Promise
  v?: Value

  // 错误状态：如果 read 抛出异常则存储在这里
  e?: AnyError
}
```

**为什么需要 `p` 集合？**
当原子 A 依赖原子 B，且 A 返回一个 Promise 时：
- 如果 B 在 Promise resolve 之前发生变化
- A 需要能够感知到这个变化并重新计算
- 即使 A 此时没有被 mount（没有监听者）
- `p` 集合确保了这种情况下的更新传播

### 1.3 Mounted 的双向依赖跟踪

```typescript
type Mounted = {
  // 监听者集合：React 组件通过 store.sub 注册的回调
  readonly l: Set<() => void>

  // 该原子依赖的其他原子（向下）
  readonly d: Set<AnyAtom>

  // 依赖该原子的其他原子（向上）
  readonly t: Set<AnyAtom>  // "t" = trackers

  // 卸载时的清理函数：来自 atom.onMount 的返回值
  u?: () => void
}
```

**双向跟踪的重要性**：
- `d` 用于挂载时递归挂载依赖
- `t` 用于：
  - 变化时向上通知所有依赖者
  - 卸载时检查是否还有其他原子依赖它

---

## 2. 版本号机制与依赖追踪

### 2.1 Epoch Number 版本号设计

Jotai 的核心创新之一是使用单调递增的版本号（epoch number）来跟踪变化：

```typescript
type EpochNumber = number

// 在 setAtomStateValueOrPromise 中
if (!hasPrevValue || !Object.is(prevValue, atomState.v)) {
  ++atomState.n  // 版本号递增！
  if (isPromiseLike(prevValue)) {
    abortPromise(store, prevValue)
  }
}
```

**版本号的三个用途**：

1. **快速检测变化**：比较版本号比深度比较值更高效
2. **依赖失效判断**：在 `readAtomState` 中：
   ```typescript
   // 检查依赖的版本号是否与记录的一致
   let hasChangedDeps = false
   for (const [a, n] of atomState.d) {
     if (readAtomState(store, a).n !== n) {
       hasChangedDeps = true
       break
     }
   }
   ```
3. **失效标记**：`invalidatedAtoms` 存储的是失效时的版本号

### 2.2 依赖追踪的完整流程

让我们通过一个具体例子来看：

```typescript
const aAtom = atom(1)
const bAtom = atom(2)
const sumAtom = atom((get) => get(aAtom) + get(bAtom))
```

当读取 `sumAtom` 时：

```
readAtomState(sumAtom)
    ↓
创建 getter 函数
    ↓
调用 sumAtom.read(getter)
    ↓
    ├─→ getter(aAtom)
    │   ↓
    │   readAtomState(aAtom)
    │   ↓
    │   返回 aAtom.v = 1
    │   ↓
    │   finally 块执行：
    │   ├─ nextDeps.set(aAtom, aAtom.n)
    │   ├─ sumAtomState.d.set(aAtom, aAtom.n)
    │   └─ if (mounted) aMounted.t.add(sumAtom)
    │
    └─→ getter(bAtom)
        ↓
        （类似 aAtom 的流程）
        ↓
        nextDeps.set(bAtom, bAtom.n)
        sumAtomState.d.set(bAtom, bAtom.n)
        if (mounted) bMounted.t.add(sumAtom)
    ↓
sumAtom.read 返回 3
    ↓
pruneDependencies():
  遍历 prevDeps（旧依赖），删除不在 nextDeps 中的
```

### 2.3 依赖修剪（Pruning）

依赖是动态的，每次 `read` 后都会修剪：

```typescript
const prevDeps = new Set<AnyAtom>(atomState.d.keys())
const nextDeps = new Map<AnyAtom, EpochNumber>()

const pruneDependencies = () => {
  for (const a of prevDeps) {
    if (!nextDeps.has(a)) {
      atomState.d.delete(a)  // 移除不再需要的依赖
    }
  }
}
```

**为什么需要修剪？**
- 条件读取场景：`get(cond) ? get(a) : get(b)`
- 避免无效更新：如果不再依赖 a，a 变化时不应触发重计算

---

## 3. 读取原子的完整流程

### 3.1 `readAtomState` 详解

这是 Jotai 最复杂的函数（200+ 行），让我们分阶段解析：

#### 阶段 1：缓存检查（Lines 520-540）

```typescript
// 检查是否可以跳过重新计算
if (isAtomStateInitialized(atomState)) {
  // 情况 A: 原子已挂载
  // 已挂载的原子会在依赖变化时即时更新，所以缓存是有效的
  if (mountedMap.has(atom) && invalidatedAtoms.get(atom) !== atomState.n) {
    return atomState  // 直接返回缓存
  }

  // 情况 B: 原子未挂载，检查依赖版本
  let hasChangedDeps = false
  for (const [a, n] of atomState.d) {
    if (readAtomState(store, a).n !== n) {
      hasChangedDeps = true
      break
    }
  }
  if (!hasChangedDeps) {
    return atomState  // 依赖未变，返回缓存
  }
}
```

**关键设计**：
- 已挂载原子可以信任缓存，因为依赖变化会立即触发失效
- 未挂载原子需要主动检查依赖版本

#### 阶段 2：准备环境（Lines 542-593）

```typescript
let isSync = true  // 标记是否在同步执行阶段
const prevDeps = new Set<AnyAtom>(atomState.d.keys())
const nextDeps = new Map<AnyAtom, EpochNumber>()

const pruneDependencies = () => { /* ... */ }
const mountDependenciesIfAsync = () => { /* ... */ }

const getter = <V>(a: Atom<V>) => {
  if (a === (atom as AnyAtom)) {
    // 自引用：get(this)
    const aState = ensureAtomState(store, a)
    if (!isAtomStateInitialized(aState)) {
      if (hasInitialValue(a)) {
        setAtomStateValueOrPromise(store, a, a.init)
      } else {
        throw new Error('no atom init')
      }
    }
    return returnAtomValue(aState)
  }

  // 读取其他原子
  const aState = readAtomState(store, a)
  try {
    return returnAtomValue(aState)
  } finally {
    // 无论成功失败都记录依赖
    nextDeps.set(a, aState.n)
    atomState.d.set(a, aState.n)

    if (isPromiseLike(atomState.v)) {
      addPendingPromiseToDependency(atom, atomState.v, aState)
    }
    if (mountedMap.has(atom)) {
      mountedMap.get(a)?.t.add(atom)
    }
    if (!isSync) {
      mountDependenciesIfAsync()
    }
  }
}
```

**`isSync` 标志的作用**：
- 同步阶段：`isSync = true`，`getter` 调用时不立即挂载依赖
- 异步阶段：`isSync = false`，`getter` 调用时需要挂载依赖

#### 阶段 3：AbortController 和 setSelf（Lines 594-633）

```typescript
let controller: AbortController | undefined
let setSelf: ((...args: unknown[]) => unknown) | undefined

const options = {
  get signal() {
    if (!controller) {
      controller = new AbortController()
    }
    return controller.signal
  },
  get setSelf() {
    // setSelf 已弃用，但为了兼容性保留
    if (!setSelf && isActuallyWritableAtom(atom)) {
      setSelf = (...args) => {
        if (!isSync) {  // 只能在异步阶段调用
          try {
            return writeAtomState(store, atom, ...args)
          } finally {
            recomputeInvalidatedAtoms(store)
            flushCallbacks(store)
          }
        }
      }
    }
    return setSelf
  },
}
```

#### 阶段 4：执行 read 并处理结果（Lines 634-673）

```typescript
const prevEpochNumber = atomState.n

try {
  const valueOrPromise = atomRead(store, atom, getter, options as never)
  setAtomStateValueOrPromise(store, atom, valueOrPromise)

  if (isPromiseLike(valueOrPromise)) {
    registerAbortHandler(store, valueOrPromise, () => controller?.abort())
    const settle = () => {
      pruneDependencies()
      mountDependenciesIfAsync()
    }
    valueOrPromise.then(settle, settle)
  } else {
    pruneDependencies()
  }

  storeHooks.r?.(atom)
  return atomState
} catch (error) {
  delete atomState.v
  atomState.e = error
  ++atomState.n
  return atomState
} finally {
  isSync = false  // 标记同步阶段结束

  // 如果版本号变化了，标记为失效和变化
  if (
    prevEpochNumber !== atomState.n &&
    invalidatedAtoms.get(atom) === prevEpochNumber
  ) {
    invalidatedAtoms.set(atom, atomState.n)
    changedAtoms.add(atom)
    storeHooks.c?.(atom)
  }
}
```

### 3.2 错误处理设计

Jotai 将错误视为一等公民：

```typescript
function returnAtomValue<Value>(atomState: AtomState<Value>): Value {
  if ('e' in atomState) {
    throw atomState.e  // 重新抛出错误
  }
  return atomState.v!
}
```

这意味着：
- 原子的 `read` 抛出的错误会被捕获并存储
- 下次读取该原子时会重新抛出错误
- 错误状态也会触发依赖更新

---

## 4. 写入与失效传播

### 4.1 `writeAtomState` 详解

```typescript
const BUILDING_BLOCK_writeAtomState: WriteAtomState = (
  store,
  atom,
  ...args
) => {
  let isSync = true

  const getter: Getter = <V>(a: Atom<V>) =>
    returnAtomValue(readAtomState(store, a))

  const setter: Setter = <V, As extends unknown[], R>(
    a: WritableAtom<V, As, R>,
    ...args: As
  ) => {
    const aState = ensureAtomState(store, a)

    try {
      if (a === (atom as AnyAtom)) {
        // 写入自身：set(this, value)
        if (!hasInitialValue(a)) {
          throw new Error('atom not writable')
        }

        const prevEpochNumber = aState.n
        const v = args[0] as V
        setAtomStateValueOrPromise(store, a, v)
        mountDependencies(store, a)

        if (prevEpochNumber !== aState.n) {
          changedAtoms.add(a)
          invalidateDependents(store, a)  // 关键：失效传播
          storeHooks.c?.(a)
        }
        return undefined as R
      } else {
        // 写入其他原子：递归调用
        return writeAtomState(store, a, ...args)
      }
    } finally {
      if (!isSync) {
        recomputeInvalidatedAtoms(store)
        flushCallbacks(store)
      }
    }
  }

  try {
    return atomWrite(store, atom, getter, setter, ...args)
  } finally {
    isSync = false
  }
}
```

### 4.2 `invalidateDependents`：失效传播

```typescript
const BUILDING_BLOCK_invalidateDependents: InvalidateDependents = (
  store,
  atom,
) => {
  const stack: AnyAtom[] = [atom]

  while (stack.length) {
    const a = stack.pop()!
    const aState = ensureAtomState(store, a)

    // 遍历所有依赖者（mounted 和 pending）
    for (const d of getMountedOrPendingDependents(a, aState, mountedMap)) {
      const dState = ensureAtomState(store, d)

      // 只在未失效时标记
      if (invalidatedAtoms.get(d) !== dState.n) {
        invalidatedAtoms.set(d, dState.n)  // 记录失效时的版本号
        stack.push(d)  // 继续向上传播
      }
    }
  }
}
```

**失效传播的特点**：
- DFS 遍历依赖图
- 每个原子只失效一次（通过版本号检查）
- `invalidatedAtoms` 存储的是失效时的版本号，用于后续判断是否需要重计算

### 4.3 一个完整的写入示例

```typescript
const aAtom = atom(1)
const bAtom = atom((get) => get(aAtom) * 2)
const cAtom = atom((get) => get(bAtom) + 1)

store.set(aAtom, 2)
```

执行流程：

```
writeAtomState(aAtom, 2)
    ↓
setAtomStateValueOrPromise(aAtom, 2)
    aAtom.n: 0 → 1
    ↓
invalidateDependents(aAtom)
    ↓
    ├─→ 找到依赖者 bAtom
    │   invalidatedAtoms.set(bAtom, bAtom.n)
    │   ↓
    │   找到依赖者 cAtom
    │   invalidatedAtoms.set(cAtom, cAtom.n)
    │
changedAtoms.add(aAtom)
    ↓
[ store.set 返回到 storeSet ]
    ↓
recomputeInvalidatedAtoms()
    ↓
flushCallbacks()
```

---

## 5. 拓扑排序重计算算法

### 5.1 为什么需要拓扑排序？

考虑这个依赖图：

```
a ← b ← c
 ↖       ↗
    d ← e
```

当 `a` 变化时，重计算顺序必须是：`a → b → d → c → e`，或者其他满足依赖关系的顺序。

如果不按拓扑顺序，可能会出现：
- 先计算 `c`，但此时 `b` 还没更新
- `c` 使用旧值，导致结果错误

### 5.2 `recomputeInvalidatedAtoms` 详解

这是 Jotai 最精妙的算法之一。

#### 步骤 1：构建反转拓扑排序（Lines 438-481）

```typescript
// 使用 DFS 进行拓扑排序
const topSortedReversed: [atom: AnyAtom, atomState: AtomState][] = []
const visiting = new WeakSet<AnyAtom>()
const visited = new WeakSet<AnyAtom>()

// 从变化的原子开始（这些是"根"原子）
const stack: AnyAtom[] = Array.from(changedAtoms)

while (stack.length) {
  const a = stack[stack.length - 1]!
  const aState = ensureAtomState(store, a)

  if (visited.has(a)) {
    stack.pop()
    continue
  }

  if (visiting.has(a)) {
    // 后序位置：所有依赖者都处理完了，现在处理这个原子
    if (invalidatedAtoms.get(a) === aState.n) {
      topSortedReversed.push([a, aState])
    }
    visited.add(a)
    stack.pop()
    continue
  }

  visiting.add(a)

  // 把依赖者压入栈
  for (const d of getMountedOrPendingDependents(a, aState, mountedMap)) {
    if (!visiting.has(d)) {
      stack.push(d)
    }
  }
}
```

**关键点**：
- 这是**后序遍历**（post-order traversal）
- 原子在所有依赖者处理完之后才加入 `topSortedReversed`
- 结果是**反转的拓扑顺序**：依赖者在前，被依赖者在后

#### 步骤 2：按拓扑顺序重计算（Lines 483-498）

```typescript
// 反向遍历，得到正确的拓扑顺序
for (let i = topSortedReversed.length - 1; i >= 0; --i) {
  const [a, aState] = topSortedReversed[i]!

  // 检查依赖是否真的变化了
  let hasChangedDeps = false
  for (const dep of aState.d.keys()) {
    if (dep !== a && changedAtoms.has(dep)) {
      hasChangedDeps = true
      break
    }
  }

  if (hasChangedDeps) {
    readAtomState(store, a)      // 重新读取
    mountDependencies(store, a)   // 更新挂载状态
  }

  invalidatedAtoms.delete(a)  // 清除失效标记
}
```

### 5.3 算法示例

假设依赖图：`a → b → c`，`a` 变化了：

```
初始 stack = [a]

迭代 1: stack = [a]
  a 未访问，未在 visiting
  visiting.add(a)
  找到 a 的依赖者 b
  stack.push(b)
  → stack = [a, b]

迭代 2: stack = [a, b]
  b 未访问，未在 visiting
  visiting.add(b)
  找到 b 的依赖者 c
  stack.push(c)
  → stack = [a, b, c]

迭代 3: stack = [a, b, c]
  c 未访问，未在 visiting
  visiting.add(c)
  c 没有依赖者
  → stack 仍为 [a, b, c]

迭代 4: stack = [a, b, c]
  c 在 visiting
  topSortedReversed.push(c)
  visited.add(c)
  stack.pop()
  → stack = [a, b], topSortedReversed = [c]

迭代 5: stack = [a, b]
  b 在 visiting
  topSortedReversed.push(b)
  visited.add(b)
  stack.pop()
  → stack = [a], topSortedReversed = [c, b]

迭代 6: stack = [a]
  a 在 visiting
  topSortedReversed.push(a)
  visited.add(a)
  stack.pop()
  → stack = [], topSortedReversed = [c, b, a]

然后反向遍历: i = 2 → 1 → 0
  i=2: a → 已变化，不需要重计算
  i=1: b → 依赖 a 已变，重计算
  i=0: c → 依赖 b 已变，重计算
```

---

## 6. 挂载生命周期管理

### 6.1 `mountAtom`：原子挂载

```typescript
const BUILDING_BLOCK_mountAtom: MountAtom = (store, atom) => {
  const atomState = ensureAtomState(store, atom)
  let mounted = mountedMap.get(atom)

  if (!mounted) {
    // 1. 先确保原子状态是最新的
    readAtomState(store, atom)

    // 2. 递归挂载依赖（先挂载依赖，再挂载自己）
    for (const a of atomState.d.keys()) {
      const aMounted = mountAtom(store, a)
      aMounted.t.add(atom)  // 建立反向跟踪
    }

    // 3. 创建挂载状态
    mounted = {
      l: new Set(),
      d: new Set(atomState.d.keys()),
      t: new Set(),
    }
    mountedMap.set(atom, mounted)

    // 4. 调用 onMount（如果有）
    if (isActuallyWritableAtom(atom)) {
      const processOnMount = () => {
        let isSync = true
        const setAtom = (...args: unknown[]) => {
          try {
            return writeAtomState(store, atom, ...args)
          } finally {
            if (!isSync) {
              recomputeInvalidatedAtoms(store)
              flushCallbacks(store)
            }
          }
        }
        try {
          const onUnmount = atomOnMount(store, atom, setAtom)
          if (onUnmount) {
            mounted!.u = () => {
              isSync = true
              try {
                onUnmount()
              } finally {
                isSync = false
              }
            }
          }
        } finally {
          isSync = false
        }
      }
      mountCallbacks.add(processOnMount)
    }

    storeHooks.m?.(atom)
  }

  return mounted
}
```

### 6.2 `unmountAtom`：原子卸载

```typescript
const BUILDING_BLOCK_unmountAtom: UnmountAtom = (store, atom) => {
  const atomState = ensureAtomState(store, atom)
  let mounted = mountedMap.get(atom)

  // 还有监听者，不能卸载
  if (!mounted || mounted.l.size) {
    return mounted
  }

  // 检查是否还有其他已挂载的原子依赖它
  let isDependent = false
  for (const a of mounted.t) {
    if (mountedMap.get(a)?.d.has(atom)) {
      isDependent = true
      break
    }
  }

  if (!isDependent) {
    // 可以安全卸载
    if (mounted.u) {
      unmountCallbacks.add(mounted.u)
    }

    mounted = undefined
    mountedMap.delete(atom)

    // 递归尝试卸载依赖
    for (const a of atomState.d.keys()) {
      const aMounted = unmountAtom(store, a)
      aMounted?.t.delete(atom)
    }

    storeHooks.u?.(atom)
    return undefined
  }

  return mounted
}
```

### 6.3 挂载状态机

```
未挂载
   ↑
   │ store.sub() 订阅
   ↓
已挂载 ←───────────┐
   ↑               │
   │ 仍有监听者     │ 所有监听者都取消订阅
   │ 或仍被依赖     │ 且没有其他原子依赖
   ↓               │
待检查 ────────────┘
```

---

## 7. React 集成的微妙之处

### 7.1 `useAtomValue` 为什么用 `useReducer`？

```typescript
const [[valueFromReducer, storeFromReducer, atomFromReducer], rerender] =
  useReducer<readonly [Value, Store, typeof atom], undefined, []>(
    (prev) => {
      const nextValue = store.get(atom)
      if (
        Object.is(prev[0], nextValue) &&
        prev[1] === store &&
        prev[2] === atom
      ) {
        return prev  // 相同引用，跳过重渲染
      }
      return [nextValue, store, atom]
    },
    undefined,
    () => [store.get(atom), store, atom],
  )
```

**为什么不用 `useState`？**

1. **`useReducer` 的惰性初始化**：第三个参数是初始化函数，只执行一次
2. **批量更新**：`useReducer` 的更新在 React 18+ 中会自动批处理
3. **引用比较**：通过 `Object.is` 比较，避免不必要的重渲染
4. **强制重渲染**：即使值相同，调用 `rerender()` 也可以强制重渲染

### 7.2 订阅的时机和顺序

```typescript
useEffect(() => {
  const unsub = store.sub(atom, () => {
    // 变化时的回调
    if (promiseStatus) {
      try {
        const value = store.get(atom)
        if (isPromiseLike(value)) {
          attachPromiseStatus(
            createContinuablePromise(store, value, () => store.get(atom)),
          )
        }
      } catch {
        // ignore
      }
    }
    if (typeof delay === 'number') {
      setTimeout(rerender, delay)
      return
    }
    rerender()
  })

  rerender()  // 关键：订阅后立即重渲染一次！
  return unsub
}, [store, atom, delay, promiseStatus])
```

**为什么订阅后要立即 `rerender()`？**

这是为了防止**时间竞争条件（time-of-check/time-of-use race）**：

```
时间线：
T1: 组件挂载，store.get(atom) → 值 A
T2: 另一个组件修改 atom → 值 B
T3: 该组件 useEffect 执行，订阅 atom
    ↓
    如果不立即 rerender，组件会一直显示 A，直到下次变化！
```

### 7.3 `Provider` 的 `storeRef` 设计

```typescript
export function Provider({ children, store }: {
  children?: ReactNode
  store?: Store
}): ReactElement {
  const storeRef = useRef<Store>(null)
  if (store) {
    // 传入了 store，直接使用
    return createElement(StoreContext.Provider, { value: store }, children)
  }
  if (storeRef.current === null) {
    // 懒创建 store
    storeRef.current = createStore()
  }
  return createElement(
    StoreContext.Provider,
    { value: storeRef.current },
    children,
  )
}
```

**为什么用 `useRef` 而不是 `useState`？**

- `useRef` 的值变化不会触发重渲染
- Store 创建后就不需要改变
- 避免不必要的 Provider 重渲染

---

## 8. 异步 Promise 处理

### 8.1 Promise 作为一等公民

Jotai 的一个核心特性是原子可以直接返回 Promise：

```typescript
const dataAtom = atom(async (get) => {
  const res = await fetch(`/api/data?id=${get(idAtom)}`)
  return res.json()
})
```

### 8.2 `createContinuablePromise`：可延续 Promise

这是 Jotai 处理异步原子的关键机制：

```typescript
const createContinuablePromise = <T>(
  store: Store,
  promise: PromiseLike<T>,
  getValue: () => PromiseLike<T> | T,
) => {
  let continuablePromise = continuablePromiseMap.get(promise)

  if (!continuablePromise) {
    continuablePromise = new Promise<T>((resolve, reject) => {
      let curr = promise

      const onFulfilled = (me: PromiseLike<T>) => (v: T) => {
        if (curr === me) {
          resolve(v)  // 仍是当前 Promise，resolve
        }
        // 否则忽略，curr 已经变了
      }

      const onRejected = (me: PromiseLike<T>) => (e: unknown) => {
        if (curr === me) {
          reject(e)
        }
      }

      const onAbort = () => {
        // Promise 被中止了（通常因为依赖变化）
        try {
          const nextValue = getValue()  // 重新读取
          if (isPromiseLike(nextValue)) {
            // 又是 Promise，继续跟踪
            continuablePromiseMap.set(nextValue, continuablePromise!)
            curr = nextValue
            nextValue.then(onFulfilled(nextValue), onRejected(nextValue))
            registerAbortHandler(store, nextValue, onAbort)
          } else {
            resolve(nextValue)  // 同步值，直接 resolve
          }
        } catch (e) {
          reject(e)
        }
      }

      promise.then(onFulfilled(promise), onRejected(promise))
      registerAbortHandler(store, promise, onAbort)
    })

    continuablePromiseMap.set(promise, continuablePromise)
  }

  return continuablePromise
}
```

### 8.3 异步竞争条件处理

考虑这个场景：

```typescript
const idAtom = atom(1)
const dataAtom = atom(async (get) => {
  const id = get(idAtom)
  const res = await fetch(`/api/data?id=${id}`)  // 假设需要 1 秒
  return res.json()
})

// 用户操作：
// T0: setId(1) → dataAtom 开始 fetch(1)
// T0.5: setId(2) → dataAtom 开始 fetch(2)
// T1: fetch(1) 返回
// T1.5: fetch(2) 返回
```

**问题**：fetch(1) 在 fetch(2) 之后返回，如何避免显示旧数据？

**Jotai 的解决方案**：

```
T0: fetch(1) 开始，创建 Promise P1
    createContinuablePromise(P1) → C1
    curr = P1

T0.5: id 变化 → invalidate dataAtom
    → readAtomState(dataAtom)
    → 发现 prevValue P1 是 Promise
    → abortPromise(store, P1)
    ↓
    fetch(2) 开始，创建 Promise P2
    createContinuablePromise(P2) → C1 (复用！)
    curr = P2
    continuablePromiseMap.set(P2, C1)

T1: P1 resolve
    onFulfilled(P1) 检查: curr === P1?
    curr 现在是 P2，不等！
    → 忽略 P1 的结果

T1.5: P2 resolve
    onFulfilled(P2) 检查: curr === P2? 是的！
    → resolve(C1) 显示正确数据
```

### 8.4 Suspense 集成

```typescript
const use =
  React.use ||
  // 旧版 React 的 shim
  (<T>(promise: PromiseLike<T> & {
    status?: 'pending' | 'fulfilled' | 'rejected'
    value?: T
    reason?: unknown
  }): T => {
    if (promise.status === 'pending') {
      throw promise  // 关键：throw Promise 触发 Suspense
    } else if (promise.status === 'fulfilled') {
      return promise.value as T
    } else if (promise.status === 'rejected') {
      throw promise.reason  // throw Error 触发 ErrorBoundary
    } else {
      attachPromiseStatus(promise)
      throw promise
    }
  })

if (isPromiseLike(value)) {
  const promise = createContinuablePromise(store, value, () => store.get(atom))
  if (promiseStatus) {
    attachPromiseStatus(promise)
  }
  return use(promise)  // 可能 throw
}
```

---

## 9. 内存管理与 GC 友好设计

### 9.1 WeakMap 的使用

Jotai 大量使用 `WeakMap` 而非普通 `Map`：

```typescript
const atomStateMap = new WeakMap<Atom, AtomState>()
const mountedMap = new WeakMap<Atom, Mounted>()
const invalidatedAtoms = new WeakMap<Atom, EpochNumber>()
```

**`WeakMap` vs `Map`**：

| 特性 | Map | WeakMap |
|------|-----|---------|
| Key 类型 | 任意 | 只能是对象 |
| Key 引用 | 强引用 | 弱引用 |
| GC 行为 | Key 不会被 GC | Key 无引用时自动回收 |
| 迭代 | 支持 | 不支持 |

**为什么这很重要？**

```typescript
function MyComponent() {
  // 在组件内部创建原子
  const localAtom = useMemo(() => atom(0), [])
  const [value, setValue] = useAtom(localAtom)

  return <div onClick={() => setValue(v => v + 1)}>{value}</div>
}

// 组件卸载后：
// - localAtom 对象没有其他引用
// - WeakMap 中的条目会被自动 GC
// - 不会内存泄漏！
```

### 9.2 `p` 集合的内存泄漏风险

回顾 `AtomState`：

```typescript
type AtomState = {
  readonly p: Set<AnyAtom>  // 注意：这是 Set，不是 WeakSet！
  // ...
}
```

**为什么用 `Set`？**

- `WeakSet` 只能存储对象，但无法遍历
- Jotai 需要在 `getMountedOrPendingDependents` 中遍历这些原子
- 权衡：功能优先，接受潜在的内存泄漏

**影响有多大？**

- 只在原子返回 Promise 且在 Promise resolve 前依赖变化时发生
- Promise resolve 后会自动清理：
  ```typescript
  const cleanup = () => dependencyAtomState.p.delete(atom)
  promise.then(cleanup, cleanup)
  ```
- 实际影响通常很小

### 9.3 挂载树的自动清理

当组件卸载时：

```
组件卸载
    ↓
useEffect cleanup 执行
    ↓
调用 store.sub() 返回的 unsub()
    ↓
从 mounted.l 中删除 listener
    ↓
调用 unmountAtom(atom)
    ↓
检查 mounted.l.size === 0?
    ↓
检查 mounted.t 中是否还有其他依赖者?
    ↓
如果都没有：
    - 调用 onUnmount 清理函数
    - 从 mountedMap 中删除
    - 递归尝试卸载依赖
```

这种设计确保了不再需要的原子会被自动清理。

---

## 总结

Jotai 的设计精巧之处在于：

1. **版本号机制**：用简单的整数实现高效的变化检测
2. **拓扑排序**：保证重计算顺序的正确性
3. **WeakMap 存储**：GC 友好的内存管理
4. **Promise 延续**：优雅处理异步竞争条件
5. **双向依赖跟踪**：精确的更新传播和生命周期管理

这些设计共同造就了 Jotai 的核心优势：小体积（~2KB）、高性能、类型安全、易于使用。
