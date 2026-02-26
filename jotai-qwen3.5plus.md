# Jotai 实现原理深度解析

> Jotai 是一个为 React 设计的原子化状态管理库，版本 2.18.0
> 核心理念：**自底向上**（bottom-up）的状态组合

---

## 目录

1. [核心架构概览](#1-核心架构概览)
2. [Atom 设计与实现](#2-atom-设计与实现)
3. [Store 内部机制](#3-store-内部机制)
4. [响应式更新流程](#4-响应式更新流程)
5. [依赖追踪系统](#5-依赖追踪系统)
6. [挂载/卸载机制](#6-挂载卸载机制)
7. [异步与 Suspense](#7-异步与-suspense)
8. [React 集成原理](#8-react-集成原理)
9. [Building Blocks 架构](#9-building-blocks-架构)
10. [高级特性](#10-高级特性)

---

## 1. 核心架构概览

### 1.1 分层架构

```
┌─────────────────────────────────────────────────────────┐
│                    React Components                      │
│  +----------------+  +----------------+                 │
│  | Counter        |  | DoubleCounter  |                 │
│  | useAtom(count) |  | useAtom(double)│                 │
│  +----------------+  +----------------+                 │
└─────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                   React Layer                            │
│  ┌─────────────┐ ┌──────────────┐ ┌─────────────────┐  │
│  │ useAtomValue│ │ useSetAtom   │ │ Provider        │  │
│  └─────────────┘ └──────────────┘ └─────────────────┘  │
└─────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                   Store Interface                        │
│  ┌──────────────────────────────────────────────────┐   │
│  │ store.get(atom) / store.set(atom, args)          │   │
│  │ store.sub(atom, listener)                        │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                Store Internals (Building Blocks)         │
│  ┌────────────────┐ ┌────────────────┐                 │
│  │ atomStateMap   │ │ mountedMap     │                 │
│  │ invalidated    │ │ changedAtoms   │                 │
│  │ readAtomState  │ │ writeAtomState │                 │
│  │ mountAtom      │ │ unmountAtom    │                 │
│  └────────────────┘ └────────────────┘                 │
└─────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                    Atom Layer                            │
│  ┌─────────────┐ ┌──────────────┐ ┌─────────────────┐  │
│  │ countAtom   │ │ doubleAtom   │ │ fetchAtom       │  │
│  │ atom(0)     │ │ atom(get=>)  │ │ atom(async=>)   │  │
│  └─────────────┘ └──────────────┘ └─────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 1.2 数据流

```
用户操作 → setAtom(value)
    │
    ▼
store.set(atom, value)
    │
    ▼
writeAtomState() ──→ 更新 atomState.v
    │
    ▼
invalidateDependents() ──→ 标记所有依赖 atom 为失效
    │
    ▼
recomputeInvalidatedAtoms() ──→ 拓扑排序后重新计算
    │
    ▼
flushCallbacks() ──→ 通知所有 listener
    │
    ▼
React rerender (useReducer rerender)
```

---

## 2. Atom 设计与实现

### 2.1 Atom 的本质

**Atom 是一个带有元数据的对象**：

```typescript
// src/vanilla/atom.ts
let keyCount = 0

export function atom<Value, Args extends unknown[], Result>(
  read?: Value | Read<Value, SetAtom<Args, unknown>>,
  write?: Write<Args, Result>,
) {
  const key = `atom${++keyCount}`  // 全局唯一 key

  const config = {
    toString() {
      return MODE !== 'production' && this.debugLabel
        ? key + ':' + this.debugLabel
        : key
    },
  } as WritableAtom<Value, Args, Result> & { init?: Value }

  if (typeof read === 'function') {
    // 派生 atom
    config.read = read as Read<Value>
  } else {
    // 原始 atom
    config.init = read  // 保存初始值
    config.read = defaultRead
    config.write = defaultWrite as unknown as Write<Args, Result>
  }

  if (write) {
    config.write = write
  }

  return config
}
```

**关键点：**
- 每个 atom 有唯一的自增 key (`atom1`, `atom2`, ...)
- `toString()` 方法在开发模式下显示 `debugLabel`
- 通过是否有 `init` 属性区分原始 atom 和派生 atom

### 2.2 Atom 的三种形态

#### 原始 Atom（Primitive Atom）

```typescript
const countAtom = atom(0)
// 等价于:
// {
//   init: 0,
//   read: (get) => get(this),  // defaultRead
//   write: (get, set, arg) => set(this, arg)  // defaultWrite
// }
```

#### 只读派生 Atom（Read-only Derived Atom）

```typescript
const doubleAtom = atom((get) => get(countAtom) * 2)
// 等价于:
// {
//   read: (get) => get(countAtom) * 2
// }
```

#### 可写派生 Atom（Writable Derived Atom）

```typescript
const decrementAtom = atom(
  (get) => get(countAtom),  // read
  (get, set) => set(countAtom, get(countAtom) - 1)  // write
)
```

### 2.3 Getter 和 Setter

```typescript
type Getter = <Value>(atom: Atom<Value>) => Value

type Setter = <Value, Args extends unknown[], Result>(
  atom: WritableAtom<Value, Args, Result>,
  ...args: Args
) => Result
```

**Getter 的深层逻辑**（在 `readAtomState` 中）：

```typescript
const getter = <V>(a: Atom<V>) => {
  if (a === atom) {
    // 获取自身值
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

  // 获取依赖 atom 的值 - 递归读取
  const aState = readAtomState(store, a)
  try {
    return returnAtomValue(aState)
  } finally {
    // 关键：记录依赖关系
    nextDeps.set(a, aState.n)  // 记录依赖的 epoch
    atomState.d.set(a, aState.n)  // 更新依赖图

    // 如果是 Promise，添加到 pending set
    if (isPromiseLike(atomState.v)) {
      addPendingPromiseToDependency(atom, atomState.v, aState)
    }

    // 建立反向依赖关系（用于通知）
    if (mountedMap.has(atom)) {
      mountedMap.get(a)?.t.add(atom)  // a.t 存储依赖 a 的 atom
    }
  }
}
```

---

## 3. Store 内部机制

### 3.1 Store 的数据结构

```typescript
// src/vanilla/internals.ts

type AtomStateMap = WeakMapLike<AnyAtom, AtomState>
type MountedMap = WeakMapLike<AnyAtom, Mounted>
type InvalidatedAtoms = WeakMapLike<AnyAtom, EpochNumber>
type ChangedAtoms = SetLike<AnyAtom>
```

### 3.2 AtomState 详解

```typescript
type AtomState<Value> = {
  /**
   * 依赖关系图
   * Key: 依赖的 atom
   * Value: 依赖 atom 的 epoch number
   *
   * 用于：
   * 1. 判断依赖是否变化
   * 2. 缓存失效判断
   */
  readonly d: Map<AnyAtom, EpochNumber>

  /**
   * 待处理的 Promise
   * 存储依赖当前 atom 的 pending atom
   *
   * 用于：
   * 1. 异步状态追踪
   * 2. Promise 取消时的清理
   *
   * 注意：可能导致内存泄漏，待优化
   */
  readonly p: Set<AnyAtom>

  /**
   * Epoch Number（版本号）
   * 每次值变化时递增
   *
   * 用于：
   * 1. 判断缓存是否有效
   * 2. 依赖变化的判断依据
   */
  n: EpochNumber

  /** Atom 的当前值 */
  v?: Value

  /** Atom 的错误状态 */
  e?: AnyError
}
```

### 3.3 Mounted 详解

```typescript
type Mounted = {
  /**
   * 监听器集合
   * React 组件的 rerender 函数存储在这里
   */
  readonly l: Set<() => void>

  /**
   * 当前 atom 依赖的其他 atom
   * 用于卸载时清理
   */
  readonly d: Set<AnyAtom>

  /**
   * 依赖当前 atom 的其他 atom（反向依赖）
   * 用于通知依赖 atom 更新
   */
  readonly t: Set<AnyAtom>

  /**
   * 卸载回调
   * 由 atom.onMount 返回
   */
  u?: () => void
}
```

### 3.4 为什么使用 WeakMap？

Jotai 使用 `WeakMap` 存储 atom 状态的原因：

1. **自动垃圾回收**：当 atom 不再被其他地方引用时，WeakMap 中的条目会被自动清理
2. **避免内存泄漏**：不需要手动清理已删除的 atom 状态
3. **性能优化**：不需要维护引用计数

---

## 4. 响应式更新流程

### 4.1 完整的更新链路

以 `setCount(c => c + 1)` 为例：

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: store.set(countAtom, prev => prev + 1)              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: writeAtomState(store, countAtom, updater)           │
│ - 创建 getter/setter 闭包                                    │
│ - 调用 atom.write(getter, setter, updater)                  │
│ - setter 中执行：setAtomStateValueOrPromise(countAtom, 1)   │
│ - countAtom.n: 0 → 1                                        │
│ - changedAtoms.add(countAtom)                               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3: invalidateDependents(store, countAtom)              │
│ - 遍历 countAtom 的所有 dependents (doubleAtom, ...)         │
│ - invalidatedAtoms.set(doubleAtom, doubleAtom.n)            │
│ - 递归标记所有传递依赖                                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 4: recomputeInvalidatedAtoms()                         │
│ - 构建拓扑排序（深度优先搜索）                               │
│ - 从后往前遍历：doubleAtom → ...                            │
│ - 对每个 atom 调用 readAtomState()                           │
│ - doubleAtom 重新计算：get(countAtom) * 2 = 2               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 5: flushCallbacks()                                    │
│ - 遍历 changedAtoms: [countAtom, doubleAtom]                │
│ - 调用 mounted.l 中的所有 listener                          │
│ - React useReducer 触发 rerender                            │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 拓扑排序算法

Jotai 使用改进的 DFS 进行拓扑排序：

```typescript
const BUILDING_BLOCK_recomputeInvalidatedAtoms = (store) => {
  const topSortedReversed: [atom: AnyAtom, atomState: AtomState][] = []
  const visiting = new WeakSet<AnyAtom>()
  const visited = new WeakSet<AnyAtom>()

  // 从 changedAtoms 开始 DFS
  const stack: AnyAtom[] = Array.from(changedAtoms)

  while (stack.length) {
    const a = stack[stack.length - 1]!
    const aState = ensureAtomState(store, a)

    if (visited.has(a)) {
      // 所有依赖已处理，可以处理当前 atom
      stack.pop()
      continue
    }

    if (visiting.has(a)) {
      // 后序遍历：依赖已处理完
      if (invalidatedAtoms.get(a) === aState.n) {
        topSortedReversed.push([a, aState])
      }
      visited.add(a)
      stack.pop()
      continue
    }

    visiting.add(a)
    // 将依赖项加入栈
    for (const d of getMountedOrPendingDependents(a, aState, mountedMap)) {
      if (!visiting.has(d)) {
        stack.push(d)
      }
    }
  }

  // 反向遍历（确保依赖先计算）
  for (let i = topSortedReversed.length - 1; i >= 0; --i) {
    const [a, aState] = topSortedReversed[i]!
    readAtomState(store, a)  // 重新计算
    mountDependencies(store, a)
    invalidatedAtoms.delete(a)
  }
}
```

---

## 5. 依赖追踪系统

### 5.1 依赖图的建立

依赖关系在 **读取时建立**，而不是在创建时：

```typescript
// 创建时
const countAtom = atom(0)
const doubleAtom = atom((get) => get(countAtom) * 2)  // 此时不记录依赖

// 读取时（readAtomState 中）
const getter = <V>(a: Atom<V>) => {
  const aState = readAtomState(store, a)  // 递归读取

  // 关键：在 finally 中建立依赖关系
  nextDeps.set(a, aState.n)  // 记录本次读取的依赖
  atomState.d.set(a, aState.n)  // 更新 atomState 的依赖图
  mountedMap.get(a)?.t.add(atom)  // 建立反向依赖
}
```

### 5.2 依赖图的更新

当 atom 的依赖发生变化时：

```typescript
// mountDependencies 函数
const BUILDING_BLOCK_mountDependencies = (store, atom) => {
  const atomState = ensureAtomState(store, atom)
  const mounted = mountedMap.get(atom)

  if (mounted) {
    // 添加新的依赖
    for (const [a, n] of atomState.d) {
      if (!mounted.d.has(a)) {
        const aState = ensureAtomState(store, a)
        const aMounted = mountAtom(store, a)
        aMounted.t.add(atom)  // 添加到反向依赖
        mounted.d.add(a)

        // 如果依赖已变化，触发更新
        if (n !== aState.n) {
          changedAtoms.add(a)
          invalidateDependents(store, a)
        }
      }
    }

    // 移除旧的依赖
    for (const a of mounted.d) {
      if (!atomState.d.has(a)) {
        mounted.d.delete(a)
        const aMounted = unmountAtom(store, a)
        aMounted?.t.delete(atom)
      }
    }
  }
}
```

### 5.3 缓存失效判断

```typescript
const BUILDING_BLOCK_readAtomState: ReadAtomState = (store, atom) => {
  const atomState = ensureAtomState(store, atom)

  // 缓存命中条件 1：atom 已初始化
  if (isAtomStateInitialized(atomState)) {
    // 缓存命中条件 2：atom 已挂载且未失效
    if (mountedMap.has(atom) && invalidatedAtoms.get(atom) !== atomState.n) {
      return atomState  // 直接使用缓存
    }

    // 缓存命中条件 3：所有依赖的 epoch 未变化
    let hasChangedDeps = false
    for (const [a, n] of atomState.d) {
      if (readAtomState(store, a).n !== n) {  // 递归检查
        hasChangedDeps = true
        break
      }
    }
    if (!hasChangedDeps) {
      return atomState  // 依赖未变化，使用缓存
    }
  }

  // 需要重新计算
  const valueOrPromise = atom.read(getter, options)
  setAtomStateValueOrPromise(store, atom, valueOrPromise)
  return atomState
}
```

---

## 6. 挂载/卸载机制

### 6.1 挂载流程

```typescript
const BUILDING_BLOCK_mountAtom: MountAtom = (store, atom) => {
  const atomState = ensureAtomState(store, atom)
  let mounted = mountedMap.get(atom)

  if (!mounted) {
    // Step 1: 重新计算 atom 状态
    readAtomState(store, atom)

    // Step 2: 先挂载依赖（确保依赖先于自身）
    for (const a of atomState.d.keys()) {
      const aMounted = mountAtom(store, a)
      aMounted.t.add(atom)  // 建立反向依赖
    }

    // Step 3: 挂载自身
    mounted = {
      l: new Set(),           // 监听器
      d: new Set(atomState.d.keys()),  // 依赖
      t: new Set(),           // 反向依赖
    }
    mountedMap.set(atom, mounted)

    // Step 4: 如果是可写 atom，调用 onMount
    if (isActuallyWritableAtom(atom)) {
      const processOnMount = () => {
        const setAtom = (...args) => writeAtomState(store, atom, ...args)
        const onUnmount = atomOnMount(store, atom, setAtom)
        if (onUnmount) {
          mounted!.u = onUnmount
        }
      }
      mountCallbacks.add(processOnMount)
    }

    storeHooks.m?.(atom)
  }

  return mounted
}
```

### 6.2 卸载流程

```typescript
const BUILDING_BLOCK_unmountAtom: UnmountAtom = (store, atom) => {
  const atomState = ensureAtomState(store, atom)
  let mounted = mountedMap.get(atom)

  // 如果未挂载或还有监听器，不卸载
  if (!mounted || mounted.l.size) {
    return mounted
  }

  // 检查是否还有依赖项依赖当前 atom
  let isDependent = false
  for (const a of mounted.t) {
    if (mountedMap.get(a)?.d.has(atom)) {
      isDependent = true
      break
    }
  }

  if (!isDependent) {
    // 卸载自身
    if (mounted.u) {
      unmountCallbacks.add(mounted.u)
    }
    mounted = undefined
    mountedMap.delete(atom)

    // 递归卸载依赖
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

### 6.3 挂载状态图

```
初始状态：
  atomStateMap: { countAtom: { d: {}, n: 0, v: 0 } }
  mountedMap: {}

useAtom(countAtom) 挂载：
  mountedMap: {
    countAtom: {
      l: {rerender},  // 组件的 rerender 函数
      d: {},          // countAtom 没有依赖
      t: {doubleAtom} // doubleAtom 依赖 countAtom
    }
  }

useAtom(doubleAtom) 挂载：
  mountedMap: {
    countAtom: {
      l: {rerender},
      d: {},
      t: {doubleAtom}
    },
    doubleAtom: {
      l: {rerender},
      d: {countAtom},  // doubleAtom 依赖 countAtom
      t: {}
    }
  }

组件卸载：
  - 如果 Counter 卸载但 DoubleCounter 还在
  - countAtom 不会被卸载（因为 doubleAtom 还依赖它）
  - 只有当所有依赖都卸载后，countAtom 才会被 GC
```

---

## 7. 异步与 Suspense

### 7.1 异步 Atom 的工作原理

```typescript
const fetchAtom = atom(async (get) => {
  const url = get(urlAtom)
  const res = await fetch(url)
  return res.json()
})
```

**读取异步 atom 的流程：**

```typescript
// readAtomState 中
const valueOrPromise = atom.read(getter, options)
setAtomStateValueOrPromise(store, atom, valueOrPromise)

if (isPromiseLike(valueOrPromise)) {
  // 注册 abort handler
  registerAbortHandler(store, valueOrPromise, () => controller?.abort())

  const settle = () => {
    pruneDependencies()
    mountDependenciesIfAsync()
  }
  valueOrPromise.then(settle, settle)  // Promise resolve 后重新计算
}
```

### 7.2 useAtomValue 中的 Promise 处理

```typescript
// src/react/useAtomValue.ts

// 1. use shim (兼容旧 React 版本)
const use =
  React.use ||
  (<T>(promise): T => {
    if (promise.status === 'pending') {
      throw promise  // 触发 Suspense
    } else if (promise.status === 'fulfilled') {
      return promise.value
    } else {
      throw promise.reason
    }
  })

// 2. attachPromiseStatus - 标记 Promise 状态
const attachPromiseStatus = (promise) => {
  if (!promise.status) {
    promise.status = 'pending'
    promise.then(
      (v) => {
        promise.status = 'fulfilled'
        promise.value = v
      },
      (e) => {
        promise.status = 'rejected'
        promise.reason = e
      },
    )
  }
}

// 3. createContinuablePromise - 处理 Promise 链
const createContinuablePromise = (store, promise, getValue) => {
  let continuablePromise = continuablePromiseMap.get(promise)

  if (!continuablePromise) {
    continuablePromise = new Promise((resolve, reject) => {
      const onAbort = () => {
        // 如果 atom 被重新触发，获取新值
        const nextValue = getValue()
        if (isPromiseLike(nextValue)) {
          continuablePromiseMap.set(nextValue, continuablePromise)
          nextValue.then(onFulfilled, onRejected)
        } else {
          resolve(nextValue)
        }
      }
      promise.then(onFulfilled, onRejected)
      registerAbortHandler(store, promise, onAbort)
    })
  }
  return continuablePromise
}

// 4. useAtomValue 主逻辑
export function useAtomValue(atom, options) {
  const store = useStore(options)

  // 使用 useReducer 触发重渲染
  const [[value], rerender] = useReducer(
    () => [store.get(atom), store, atom],
    undefined,
    () => [store.get(atom), store, atom]
  )

  // 订阅变化
  useEffect(() => {
    const unsub = store.sub(atom, () => {
      rerender()
    })
    return unsub
  }, [store, atom])

  const value = store.get(atom)

  // 如果是 Promise，使用 use() 触发 Suspense
  if (isPromiseLike(value)) {
    const promise = createContinuablePromise(store, value, () =>
      store.get(atom)
    )
    attachPromiseStatus(promise)
    return use(promise)  // 抛出 Promise 触发 Suspense
  }

  return value
}
```

### 7.3 AbortController 支持

```typescript
// 在 readAtomState 中
let controller: AbortController | undefined
const options = {
  get signal() {
    if (!controller) {
      controller = new AbortController()
    }
    return controller.signal
  }
}

// Promise 被取消时
const valueOrPromise = atom.read(getter, { signal: options.signal })

if (isPromiseLike(valueOrPromise)) {
  registerAbortHandler(store, valueOrPromise, () => controller?.abort())
}
```

---

## 8. React 集成原理

### 8.1 useAtom 的组合实现

```typescript
// src/react/useAtom.ts
export function useAtom(atom, options) {
  return [
    useAtomValue(atom, options),  // 读取值
    useSetAtom(atom, options),    // 获取 setter
  ]
}
```

### 8.2 useAtomValue 深度解析

```typescript
// src/react/useAtomValue.ts

export function useAtomValue(atom, options) {
  const store = useStore(options)

  // 使用 useReducer 而不是 useState 的原因：
  // 1. 可以自定义 reducer 逻辑
  // 2. 可以批量更新

  const [[valueFromReducer, storeFromReducer, atomFromReducer], rerender] =
    useReducer(
      (prev) => {
        const nextValue = store.get(atom)

        // 优化：如果值没变，不重渲染
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
      () => [store.get(atom), store, atom]  // 初始值
    )

  let value = valueFromReducer

  // 如果 store 或 atom 变化，立即重新获取
  if (storeFromReducer !== store || atomFromReducer !== atom) {
    rerender()
    value = store.get(atom)
  }

  // 订阅 atom 变化
  useEffect(() => {
    const unsub = store.sub(atom, () => {
      // 订阅回调：触发重渲染
      rerender()
    })
    return unsub
  }, [store, atom])

  return value
}
```

### 8.3 useSetAtom 的实现

```typescript
// src/react/useSetAtom.ts
export function useSetAtom(atom, options) {
  const store = useStore(options)

  const setAtom = useCallback(
    (...args) => {
      if (MODE !== 'production' && !('write' in atom)) {
        throw new Error('not writable atom')
      }
      return store.set(atom, ...args)
    },
    [store, atom]  // 依赖 store 和 atom
  )

  return setAtom
}
```

### 8.4 Provider 的实现

```typescript
// src/react/Provider.ts
const StoreContext = createContext<Store | undefined>(undefined)

export function Provider({ children, store }) {
  const storeRef = useRef<Store>(null)

  if (store) {
    // 传入了自定义 store
    return createElement(StoreContext.Provider, { value: store }, children)
  }

  // 未传入 store，创建默认 store（单例模式）
  if (storeRef.current === null) {
    storeRef.current = createStore()
  }

  return createElement(
    StoreContext.Provider,
    { value: storeRef.current },
    children
  )
}

export function useStore(options) {
  const store = useContext(StoreContext)
  return options?.store || store || getDefaultStore()
}
```

---

## 9. Building Blocks 架构

### 9.1 Building Blocks 的设计思想

Jotai 使用**Building Blocks**模式将核心功能拆分为 28 个可组合的函数：

```typescript
type BuildingBlocks = [
  // Store State (0-6)
  atomStateMap,           // 0: 存储 atom 状态
  mountedMap,             // 1: 存储挂载信息
  invalidatedAtoms,       // 2: 失效标记
  changedAtoms,           // 3: 变更集合
  mountCallbacks,         // 4: 挂载回调队列
  unmountCallbacks,       // 5: 卸载回调队列
  storeHooks,             // 6: 生命周期钩子

  // Atom Interceptors (7-10)
  atomRead,               // 7: 读取拦截器
  atomWrite,              // 8: 写入拦截器
  atomOnInit,             // 9: 初始化钩子
  atomOnMount,            // 10: 挂载钩子

  // Building Block Functions (11-20)
  ensureAtomState,        // 11: 确保 atom 状态存在
  flushCallbacks,         // 12: 刷新回调
  recomputeInvalidatedAtoms, // 13: 重新计算失效 atom
  readAtomState,          // 14: 读取 atom 状态
  invalidateDependents,   // 15: 标记依赖失效
  writeAtomState,         // 16: 写入 atom 状态
  mountDependencies,      // 17: 挂载依赖
  mountAtom,              // 18: 挂载 atom
  unmountAtom,            // 19: 卸载 atom
  setAtomStateValueOrPromise, // 20: 设置值或 Promise

  // Store API (21-23)
  storeGet,               // 21: get 方法
  storeSet,               // 22: set 方法
  storeSub,               // 23: sub 方法

  // Extensions (24)
  enhanceBuildingBlocks,  // 24: 扩展构建块

  // Abort Support (25-27)
  abortHandlersMap,       // 25: Abort handler 映射
  registerAbortHandler,   // 26: 注册 abort handler
  abortPromise,           // 27: 触发 abort
]
```

### 9.2 Building Blocks 的使用

```typescript
// Store 的实现
function buildStore(...buildArgs): Store {
  const store = {
    get(atom) {
      const storeGet = getInternalBuildingBlocks(store)[21]
      return storeGet(store, atom)
    },
    set(atom, ...args) {
      const storeSet = getInternalBuildingBlocks(store)[22]
      return storeSet(store, atom, ...args)
    },
    sub(atom, listener) {
      const storeSub = getInternalBuildingBlocks(store)[23]
      return storeSub(store, atom, listener)
    },
  }

  const buildingBlocks = [...]  // 28 个构建块
  buildingBlockMap.set(store, Object.freeze(buildingBlocks))
  return store
}

// 使用示例：读取 atom
const value = store.get(atom)
// 实际调用：BUILDING_BLOCK_storeGet(store, atom)
// → BUILDING_BLOCK_readAtomState(store, atom)
// → atom.read(getter, options)
```

### 9.3 扩展 Building Blocks

通过 `enhanceBuildingBlocks` 可以扩展 Jotai 的功能：

```typescript
// 示例：添加日志
const enhanceBuildingBlocks = (blocks) => {
  const originalRead = blocks[14]  // readAtomState
  return Object.freeze([
    ...blocks,
    [14]: (store, atom) => {
      console.log('Reading atom:', atom)
      return originalRead(store, atom)
    }
  ])
}
```

---

## 10. 高级特性

### 10.1 Atom Family

```typescript
// src/vanilla/utils/atomFamily.ts
export function atomFamily<Param, AtomType extends Atom<unknown>>(
  initializeAtom: (param: Param) => AtomType,
) {
  const atomMap = new Map<Param, AtomType>()
  const listeners = new Set<Listener>()

  return (param: Param): AtomType => {
    let atom = atomMap.get(param)
    if (!atom) {
      atom = initializeAtom(param)
      atomMap.set(param, atom)
    }
    return atom as AtomType
  }
}

// 使用示例
const todoAtom = atomFamily((id: number) =>
  atom({ id, text: '', completed: false })
)

// 在组件中
const todo1 = useAtom(todoAtom(1))
const todo2 = useAtom(todoAtom(2))
```

### 10.2 Loadable

```typescript
// src/vanilla/utils/loadable.ts
type Loadable<Value> =
  | { state: 'loading' }
  | { state: 'hasError'; error: unknown }
  | { state: 'hasData'; data: Value }

export function loadable<Value>(anAtom: Atom<Value>): Atom<Loadable<Value>> {
  return atom((get) => {
    try {
      return { state: 'hasData', data: get(anAtom) as Value }
    } catch (error) {
      if (error instanceof Promise) {
        return { state: 'loading' }
      }
      return { state: 'hasError', error }
    }
  })
}

// 使用示例
const fetchAtom = atom(async () => fetch('/api').then(r => r.json()))
const loadableAtom = loadable(fetchAtom)

function Component() {
  const [loadable] = useAtom(loadableAtom)

  if (loadable.state === 'loading') return <div>Loading...</div>
  if (loadable.state === 'hasError') return <div>Error: {loadable.error}</div>
  return <div>Data: {loadable.data}</div>
}
```

### 10.3 Select Atom（部分选择）

```typescript
// src/vanilla/utils/selectAtom.ts
export function selectAtom<Value, Slice>(
  anAtom: Atom<Value>,
  selector: (v: Value) => Slice,
  equalityFn: (a: Slice, b: Slice) => boolean = Object.is,
): Atom<Slice> {
  return atom(
    (get) => {
      const value = get(anAtom)
      return selector(value)
    }
  )
}

// 使用示例
const userAtom = atom({ name: 'John', age: 30, email: 'john@example.com' })
const nameAtom = selectAtom(userAtom, (user) => user.name)

// 只有 name 变化时组件才会重渲染
const [name] = useAtom(nameAtom)
```

### 10.4 Atom with Storage

```typescript
// src/vanilla/utils/atomWithStorage.ts
export function atomWithStorage<Value>(
  key: string,
  initialValue: Value,
  storage: Storage = localStorage,
) {
  return atom(
    (get) => {
      const item = storage.getItem(key)
      if (item === null) {
        return initialValue
      }
      return JSON.parse(item) as Value
    },
    (get, set, update: Value | ((prev: Value) => Value)) => {
      const nextValue =
        typeof update === 'function'
          ? (update as (prev: Value) => Value)(get(anAtom))
          : update

      storage.setItem(key, JSON.stringify(nextValue))
      set(anAtom, nextValue)
    }
  )
}
```

---

## 11. 性能优化

### 11.1 缓存策略

Jotai 使用多级缓存策略：

1. **AtomState 缓存**：每个 atom 的值被缓存
2. **依赖关系缓存**：依赖图被缓存在 `atomState.d` 中
3. **挂载状态缓存**：mounted atom 被缓存在 `mountedMap` 中

### 11.2 避免重渲染

```typescript
// useAtomValue 中的优化
const [[value], rerender] = useReducer(
  (prev) => {
    const nextValue = store.get(atom)

    // 使用 Object.is 进行浅比较
    if (Object.is(prev[0], nextValue)) {
      return prev  // 值没变，不重渲染
    }
    return [nextValue, store, atom]
  },
  ...
)
```

### 11.3 惰性计算

```typescript
// 只有 mounted 的 atom 才会被计算
const BUILDING_BLOCK_mountAtom = (store, atom) => {
  if (!mounted) {
    readAtomState(store, atom)  // 挂载时才计算
    ...
  }
}
```

---

## 12. 总结

### 12.1 核心设计模式

| 模式 | 应用 |
|------|------|
| Observer | store.sub 订阅通知 |
| Dependency Injection | Getter/Setter 注入 |
| Lazy Evaluation | 惰性计算 atom 值 |
| Topological Sort | 依赖更新顺序 |
| WeakMap | 自动垃圾回收 |

### 12.2 与其他状态管理库的对比

| 特性 | Jotai | Recoil | Zustand | Redux |
|------|-------|--------|---------|-------|
| 核心大小 | ~2KB | ~14KB | ~1KB | ~2KB |
| 依赖追踪 | 自动 | 自动 | 手动 | 手动 |
| 原子化 | 是 | 是 | 否 | 否 |
| GC 支持 | 是 | 是 | 否 | 否 |
| TypeScript | 完整 | 完整 | 完整 | 需配置 |
| 学习曲线 | 低 | 中 | 低 | 高 |

### 12.3 适用场景

- ✅ 细粒度状态更新
- ✅ 组件树外的状态共享
- ✅ 复杂派生状态
- ✅ 服务端渲染
- ❌ 需要时间旅行调试
- ❌ 需要中间件生态

---

## 附录：关键代码位置

```
src/
├── vanilla/
│   ├── atom.ts           # Atom 定义（~130 行）
│   ├── store.ts          # Store 创建（~40 行）
│   ├── internals.ts      # 核心实现（~1080 行）
│   └── utils/            # 工具函数
├── react/
│   ├── Provider.ts       # Context Provider（~50 行）
│   ├── useAtom.ts        # useAtom hook（~60 行）
│   ├── useAtomValue.ts   # 只读 hook（~180 行）
│   ├── useSetAtom.ts     # 只写 hook（~40 行）
│   └── utils/            # 工具 hooks
└── babel/                # Babel 插件
```
