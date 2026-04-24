## Jotai 实现原理（深度分析）

Jotai 是一个轻量级的 React 状态管理库，核心代码约 1000 行。其架构分为两层：**vanilla**（纯 JS 状态引擎，不依赖 React）和 **react**（React 绑定层）。

---

### 1. Atom — 配置对象，而非状态容器

`src/vanilla/atom.ts`

`atom()` 的返回值**只是一个配置对象**，不持有任何运行时状态：

```ts
// 全局递增计数器，保证每个 atom 有唯一 key
let keyCount = 0

export function atom(read?, write?) {
  const key = `atom${++keyCount}`
  const config = {
    toString() { return this.debugLabel ? key + ':' + this.debugLabel : key },
  }
  if (typeof read === 'function') {
    config.read = read       // 派生 atom：用户提供的 read 函数
  } else {
    config.init = read       // 原始 atom：read 参数实际上是初始值
    config.read = defaultRead
    config.write = defaultWrite
  }
  if (write) config.write = write
  return config
}
```

**原始 atom 的默认 read/write**：

```ts
// defaultRead: get(this) → 读取自身在 store 中的值
function defaultRead(this: Atom, get: Getter) {
  return get(this)
}

// defaultWrite: set(this, newValue) → 写入自身，支持函数式更新
function defaultWrite(this: PrimitiveAtom, get, set, arg) {
  return set(this, typeof arg === 'function' ? arg(get(this)) : arg)
}
```

这里有一个精妙之处：`defaultRead` 调用 `get(this)`，而 `get` 在 store 内部会判断 `a === atom`（读自身），直接返回存储的值而不会递归。`defaultWrite` 调用 `set(this, value)`，在 store 内部同样有 `a === atom` 的特殊处理，直接设置值而非递归调 write。

**Atom 的类型体系**（4 种重载签名）：

| 调用方式 | 返回类型 | 含义 |
|---------|---------|------|
| `atom(initialValue)` | `PrimitiveAtom<V>` | 原始可读写 atom |
| `atom()` | `PrimitiveAtom<V \| undefined>` | 无初始值的原始 atom |
| `atom((get) => ...)` | `Atom<V>` | 只读派生 atom |
| `atom(read, write)` | `WritableAtom<V, Args, Result>` | 可读写派生 atom |
| `atom(initialValue, write)` | `WritableAtom` + `WithInitialValue` | 带初始值的写 atom |

**关键设计决策**：atom 是一个身份标识（identity key），用对象引用作为 WeakMap 的 key。同一个 atom 对象在不同 store 中可以有不同的值，这让 Provider 隔离成为可能。

---

### 2. Store 与 BuildingBlocks — 可插拔的状态引擎

`src/vanilla/store.ts` + `src/vanilla/internals.ts`

#### 2.1 Store 的极简公共 API

```ts
type Store = {
  get: <V>(atom: Atom<V>) => V
  set: <V, Args, Result>(atom: WritableAtom<V, Args, Result>, ...args: Args) => Result
  sub: (atom: AnyAtom, listener: () => void) => () => void
}
```

仅 3 个方法，但内部实现由 **BuildingBlocks** 驱动——一个 28 元素的元组：

```ts
type BuildingBlocks = [
  // ===== 存储状态（6 个数据结构） =====
  atomStateMap: WeakMap<Atom, AtomState>,      // [0] atom → 状态（值/错误/依赖/epoch）
  mountedMap: WeakMap<Atom, Mounted>,           // [1] atom → 挂载信息
  invalidatedAtoms: WeakMap<Atom, EpochNumber>, // [2] 被标记失效的 atom
  changedAtoms: Set<Atom>,                      // [3] 本轮变更的 atom
  mountCallbacks: Set<() => void>,              // [4] 待执行的 onMount 回调
  unmountCallbacks: Set<() => void>,            // [5] 待执行的 onUnmount 回调
  storeHooks: StoreHooks,                       // [6] 生命周期钩子（实验性）

  // ===== Atom 拦截器（4 个，可替换 atom 行为） =====
  atomRead: AtomRead,                           // [7]  拦截 atom.read 调用
  atomWrite: AtomWrite,                         // [8]  拦截 atom.write 调用
  atomOnInit: AtomOnInit,                       // [9]  拦截 atom 首次初始化
  atomOnMount: AtomOnMount,                     // [10] 拦截 atom.onMount

  // ===== 构建块函数（10 个核心函数） =====
  ensureAtomState,                              // [11] 确保 atom 有 state（惰性创建）
  flushCallbacks,                               // [12] 批量执行监听器回调
  recomputeInvalidatedAtoms,                    // [13] 拓扑排序重算失效 atom
  readAtomState,                                // [14] 读取/计算 atom 状态
  invalidateDependents,                         // [15] 向上传播失效标记
  writeAtomState,                               // [16] 写入 atom 状态
  mountDependencies,                            // [17] 同步挂载依赖关系
  mountAtom,                                    // [18] 挂载 atom（订阅时触发）
  unmountAtom,                                  // [19] 卸载 atom
  setAtomStateValueOrPromise,                   // [20] 设置值或 Promise

  // ===== Store 顶层 API 实现 =====
  storeGet,                                     // [21] store.get 实现
  storeSet,                                     // [22] store.set 实现
  storeSub,                                     // [23] store.sub 实现
  enhanceBuildingBlocks,                        // [24] 扩展点（devtools 用）

  // ===== 可中断 Promise 支持 =====
  abortHandlersMap: WeakMap,                    // [25] promise → abort handlers
  registerAbortHandler,                         // [26] 注册中断回调
  abortPromise,                                 // [27] 触发中断
]
```

#### 2.2 BuildingBlocks 的可插拔设计

`buildStore` 接受一个 `Partial<BuildingBlocks>` 参数，允许替换任意构建块：

```ts
function buildStore(...buildArgs: Partial<BuildingBlocks>): Store {
  const store = { get, set, sub }  // 闭包引用 buildingBlocks
  const buildingBlocks = defaultBlocks.map((fn, i) => buildArgs[i] || fn)
  buildingBlockMap.set(store, Object.freeze(buildingBlocks))
  return store
}
```

这意味着你可以替换 `atomRead` 来拦截所有 atom 读取（如 devtools 的日志），替换 `readAtomState` 来改变缓存策略，或替换 `flushCallbacks` 来改变通知时机。`enhanceBuildingBlocks`（索引 24）提供了另一个扩展点，允许外部代码包装整个 BuildingBlocks。

#### 2.3 AtomState — 单个 atom 的运行时状态

```ts
type AtomState<Value> = {
  readonly d: Map<AnyAtom, EpochNumber>  // 依赖图：我依赖的 atom → 它的 epoch
  readonly p: Set<AnyAtom>               // pending：等待本 atom promise 解决的其他 atom
  n: EpochNumber                         // 本 atom 的 epoch 编号（每次值变更 +1）
  v?: Value                              // 当前值
  e?: AnyError                           // 当前错误
}
```

**Epoch 机制详解**：`n` 是一个单调递增整数。每当 atom 的值发生变化（`!Object.is(prev, next)`），epoch 就递增。依赖图 `d` 中记录的是**依赖被读取时的 epoch**。后续校验时只需比较 `d.get(dep) !== dep.n` 即可判断依赖是否过期，比深比较 O(1) 且零分配。

---

### 3. 依赖追踪 — readAtomState 的核心逻辑

`readAtomState`（internals.ts:504-674）是整个 Jotai 最复杂的函数，约 170 行。

#### 3.1 缓存命中判断（三级策略）

```ts
if (isAtomStateInitialized(atomState)) {
  // 策略 1：已挂载且未失效 → 直接返回缓存
  // 因为挂载的 atom 的依赖变更会实时推送并标记 invalidated
  if (mountedMap.has(atom) && invalidatedAtoms.get(atom) !== atomState.n) {
    return atomState  // ← 热路径，O(1)
  }

  // 策略 2：未挂载，递归检查依赖 epoch 是否一致
  let hasChangedDeps = false
  for (const [a, n] of atomState.d) {
    if (readAtomState(store, a).n !== n) {  // ← 递归！拉模式
      hasChangedDeps = true
      break
    }
  }
  if (!hasChangedDeps) return atomState  // 依赖没变 → 缓存有效
}
// 策略 3：缓存无效或未初始化 → 重新计算
```

**推拉混合模型**：
- **推模式（Push）**：已挂载的 atom，依赖变更时通过 `invalidateDependents` 主动标记失效，读取时只需检查标记
- **拉模式（Pull）**：未挂载的 atom，读取时递归检查依赖链是否过期

这是一个性能关键的设计：已订阅（挂载）的 atom 享受 O(1) 缓存命中，而未订阅的 atom 通过惰性拉取避免不必要的计算。

#### 3.2 getter 闭包 — 依赖收集的核心

```ts
const getter = <V>(a: Atom<V>) => {
  if (a === (atom as AnyAtom)) {
    // 读自身：直接返回 init 或已有值，不创建自依赖
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

  // 读其他 atom：递归计算 + 记录依赖
  const aState = readAtomState(store, a)   // 递归！
  try {
    return returnAtomValue(aState)
  } finally {
    // ① 记录依赖关系
    nextDeps.set(a, aState.n)
    atomState.d.set(a, aState.n)

    // ② 如果当前 atom 的值是 Promise，注册 pending 关系
    if (isPromiseLike(atomState.v)) {
      addPendingPromiseToDependency(atom, atomState.v, aState)
    }

    // ③ 如果当前 atom 已挂载，维护反向依赖
    if (mountedMap.has(atom)) {
      mountedMap.get(a)?.t.add(atom)
    }

    // ④ 异步场景：新增的依赖需要立即挂载
    if (!isSync) {
      mountDependenciesIfAsync()
    }
  }
}
```

**`isSync` 标志**：在 `atom.read(getter, options)` 调用期间 `isSync = true`，调用结束后设为 `false`。这区分了：
- **同步依赖**：在 read 函数同步执行期间通过 `get()` 收集的依赖，read 结束后统一 `pruneDependencies`
- **异步依赖**：在 async read 的 `await` 之后通过 `get()` 收集的依赖，需要**立即**挂载并触发重算，因为此时外层的 flush 循环已经结束

#### 3.3 依赖修剪

```ts
const prevDeps = new Set(atomState.d.keys())
const nextDeps = new Map()

// ... getter 执行期间填充 nextDeps ...

const pruneDependencies = () => {
  for (const a of prevDeps) {
    if (!nextDeps.has(a)) {
      atomState.d.delete(a)  // 不再依赖 → 移除
    }
  }
}
```

条件依赖（`get(condition) ? get(a) : get(b)`）在每次重算后会自动修剪不再需要的分支。

#### 3.4 AbortController 支持

```ts
const options = {
  get signal() {
    if (!controller) controller = new AbortController()
    return controller.signal
  },
  // ...
}
```

`signal` 是惰性创建的（只在 read 函数访问时才创建 AbortController）。当 atom 的值被新的 Promise 替换时（`setAtomStateValueOrPromise` 中 epoch 递增），旧 Promise 的 abort handler 会被触发：

```ts
// setAtomStateValueOrPromise 中：
if (!hasPrevValue || !Object.is(prevValue, atomState.v)) {
  ++atomState.n
  if (isPromiseLike(prevValue)) {
    abortPromise(store, prevValue)  // 中断旧 Promise
  }
}
```

这让 async atom 可以监听 `signal.aborted` 来取消进行中的请求。

---

### 4. 变更传播 — 写入路径的完整流程

#### 4.1 writeAtomState（internals.ts:698-757）

```ts
const BUILDING_BLOCK_writeAtomState = (store, atom, ...args) => {
  let isSync = true
  const getter = (a) => returnAtomValue(readAtomState(store, a))

  const setter = (a, ...args) => {
    const aState = ensureAtomState(store, a)
    try {
      if (a === atom) {
        // 写自身（原始 atom 的典型场景）
        const prevEpochNumber = aState.n
        setAtomStateValueOrPromise(store, a, args[0])
        mountDependencies(store, a)
        if (prevEpochNumber !== aState.n) {
          changedAtoms.add(a)
          invalidateDependents(store, a)    // 向上传播失效
        }
        return undefined
      } else {
        // 写其他 atom → 递归调 writeAtomState
        return writeAtomState(store, a, ...args)
      }
    } finally {
      if (!isSync) {
        recomputeInvalidatedAtoms(store)    // 异步写入后立即重算
        flushCallbacks(store)               // 并通知监听器
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

**setter 的 `a === atom` 分支**是批量更新的关键：一个 write 函数中可以 `set(otherAtom1, v1); set(otherAtom2, v2);`，所有写入都在同步阶段完成，`isSync = true` 时不触发 recompute/flush。直到 `atomWrite` 返回后 `isSync = false`，由外层 `storeSet` 统一调用一次 `recomputeInvalidatedAtoms` + `flushCallbacks`。

#### 4.2 invalidateDependents（internals.ts:676-696）— BFS 失效传播

```ts
const BUILDING_BLOCK_invalidateDependents = (store, atom) => {
  const stack = [atom]
  while (stack.length) {
    const a = stack.pop()!
    const aState = ensureAtomState(store, a)
    for (const d of getMountedOrPendingDependents(a, aState, mountedMap)) {
      const dState = ensureAtomState(store, d)
      if (invalidatedAtoms.get(d) !== dState.n) {
        invalidatedAtoms.set(d, dState.n)   // 标记失效（记录失效时的 epoch）
        stack.push(d)                        // 继续向上传播
      }
    }
  }
}
```

只标记不计算，标记使用 `invalidatedAtoms.set(d, dState.n)` 记录**失效时**的 epoch，避免重复标记。

`getMountedOrPendingDependents` 收集两类依赖者：
- `mountedMap.get(atom)?.t` — 已挂载的反向依赖
- `atomState.p` — 有 pending promise 依赖本 atom 的 atom（即使未挂载，promise 链也需要更新）

#### 4.3 recomputeInvalidatedAtoms（internals.ts:428-499）— 拓扑排序重算

这是整个传播机制中最精巧的部分，分两步：

**Step 1：DFS 拓扑排序**
```ts
const topSortedReversed = []
const visiting = new WeakSet()
const visited = new WeakSet()
const stack = Array.from(changedAtoms)  // 从变更源开始

while (stack.length) {
  const a = stack[stack.length - 1]!
  const aState = ensureAtomState(store, a)

  if (visited.has(a)) { stack.pop(); continue }

  if (visiting.has(a)) {
    // 后序遍历：所有子节点已处理，现在处理自己
    if (invalidatedAtoms.get(a) === aState.n) {
      topSortedReversed.push([a, aState])  // 反向插入
    }
    visited.add(a)
    stack.pop()
    continue
  }

  visiting.add(a)
  // 将未访问的依赖者压栈
  for (const d of getMountedOrPendingDependents(a, aState, mountedMap)) {
    if (!visiting.has(d)) stack.push(d)
  }
}
```

**Step 2：按拓扑序重算**
```ts
for (let i = topSortedReversed.length - 1; i >= 0; --i) {
  const [a, aState] = topSortedReversed[i]!
  // 短路优化：如果实际依赖都没变，跳过重算
  let hasChangedDeps = false
  for (const dep of aState.d.keys()) {
    if (dep !== a && changedAtoms.has(dep)) {
      hasChangedDeps = true
      break
    }
  }
  if (hasChangedDeps) {
    readAtomState(store, a)          // 重新计算
    mountDependencies(store, a)      // 同步挂载关系
  }
  invalidatedAtoms.delete(a)
}
```

**为什么不需要检测循环？** 注释说"we don't bother to check for cycles"——因为 `changedAtoms` 只包含根变更源，拓扑排序从源头向依赖者遍历，循环依赖在 `visiting` 检查中会被自然跳过（已在 visiting 中的不会再压栈）。

**短路优化**的作用：假设 A → B → D，A → C → D，如果 A 变了但 B 重算后值不变，那 D 的依赖检查发现 `changedAtoms` 中只有 A（B 没变），如果 D 不直接依赖 A，就可以跳过重算。

#### 4.4 flushCallbacks（internals.ts:390-426）— 批量通知

```ts
const BUILDING_BLOCK_flushCallbacks = (store) => {
  const errors = []
  const call = (fn) => { try { fn() } catch (e) { errors.push(e) } }

  do {
    storeHooks.f?.()                           // 触发 flush 钩子
    const callbacks = new Set()
    changedAtoms.forEach(atom =>
      mountedMap.get(atom)?.l.forEach(cb => callbacks.add(cb))
    )
    changedAtoms.clear()
    unmountCallbacks.forEach(cb => callbacks.add(cb))
    unmountCallbacks.clear()
    mountCallbacks.forEach(cb => callbacks.add(cb))
    mountCallbacks.clear()
    callbacks.forEach(call)

    // 如果回调中又触发了新的变更，继续循环
    if (changedAtoms.size) {
      recomputeInvalidatedAtoms(store)
    }
  } while (changedAtoms.size || unmountCallbacks.size || mountCallbacks.size)

  if (errors.length) throw new AggregateError(errors)
}
```

关键细节：
1. **使用 `do...while` 循环**：回调中可能触发 `set`，产生新的 `changedAtoms`，需要继续处理直到稳定
2. **AggregateError 收集**：不因单个 listener 报错而中断其他 listener
3. **执行顺序**：先 changed listeners，再 unmount callbacks，最后 mount callbacks

---

### 5. 挂载/卸载机制 — 精确的订阅管理

#### 5.1 Mounted 数据结构

```ts
type Mounted = {
  readonly l: Set<() => void>   // listeners：直接订阅本 atom 的回调
  readonly d: Set<AnyAtom>      // dependencies：本 atom 已挂载的依赖
  readonly t: Set<AnyAtom>      // dependents（反向）：已挂载的、依赖本 atom 的 atom
  u?: () => void                // unmount 回调
}
```

`d`（正向依赖）和 `t`（反向依赖）构成了一个**双向图**，使得失效传播和卸载都可以高效遍历。

#### 5.2 mountAtom（internals.ts:794-857）

```ts
const BUILDING_BLOCK_mountAtom = (store, atom) => {
  let mounted = mountedMap.get(atom)
  if (!mounted) {
    readAtomState(store, atom)          // 确保值已计算

    // 先递归挂载所有依赖
    for (const a of atomState.d.keys()) {
      const aMounted = mountAtom(store, a)
      aMounted.t.add(atom)              // 在依赖的 t 集合中注册自己
    }

    // 创建 Mounted 结构
    mounted = {
      l: new Set(),
      d: new Set(atomState.d.keys()),
      t: new Set(),
    }
    mountedMap.set(atom, mounted)

    // 如果 atom 定义了 onMount，延迟到 flushCallbacks 时执行
    if (isActuallyWritableAtom(atom)) {
      const processOnMount = () => {
        let isSync = true
        const setAtom = (...args) => {
          try { return writeAtomState(store, atom, ...args) }
          finally {
            if (!isSync) {
              recomputeInvalidatedAtoms(store)
              flushCallbacks(store)
            }
          }
        }
        try {
          const onUnmount = atomOnMount(store, atom, setAtom)
          if (onUnmount) mounted.u = onUnmount  // 保存卸载回调
        } finally { isSync = false }
      }
      mountCallbacks.add(processOnMount)
    }
  }
  return mounted
}
```

`onMount` 的设计：
- 在 mount 时接收一个 `setAtom` 函数，可以立即设置 atom 值（如发起请求）
- 返回一个 `onUnmount` 函数，在卸载时调用（如取消请求）
- **延迟执行**：mount 回调被加入 `mountCallbacks`，在 `flushCallbacks` 中统一执行，避免在挂载过程中产生副作用

#### 5.3 unmountAtom（internals.ts:859-894）— 精确的引用计数

```ts
const BUILDING_BLOCK_unmountAtom = (store, atom) => {
  let mounted = mountedMap.get(atom)
  if (!mounted || mounted.l.size) return mounted  // 还有 listener → 不卸载

  // 检查是否还有已挂载的 atom 依赖本 atom
  let isDependent = false
  for (const a of mounted.t) {
    if (mountedMap.get(a)?.d.has(atom)) {
      isDependent = true
      break
    }
  }

  if (!isDependent) {
    // 安全卸载
    if (mounted.u) unmountCallbacks.add(mounted.u)
    mountedMap.delete(atom)
    // 递归卸载依赖
    for (const a of atomState.d.keys()) {
      const aMounted = unmountAtom(store, a)
      aMounted?.t.delete(atom)
    }
  }
  return mounted
}
```

卸载条件必须同时满足：
1. `mounted.l.size === 0` — 没有直接 listener
2. 没有任何已挂载的 atom 在其 `d`（已挂载依赖）中包含本 atom

这实现了**精确的引用计数式 GC**，不多不少。

#### 5.4 mountDependencies（internals.ts:759-792）— 同步依赖图

当一个已挂载的 atom 被重新计算后，其依赖可能变了（条件 `get`），需要同步挂载关系：

```ts
// 新增的依赖 → 挂载
for (const [a, n] of atomState.d) {
  if (!mounted.d.has(a)) {
    const aMounted = mountAtom(store, a)
    aMounted.t.add(atom)
    mounted.d.add(a)
    // 如果新依赖的 epoch 已经变了，需要标记变更
    if (n !== aState.n) {
      changedAtoms.add(a)
      invalidateDependents(store, a)
    }
  }
}

// 移除的依赖 → 卸载
for (const a of mounted.d) {
  if (!atomState.d.has(a)) {
    mounted.d.delete(a)
    const aMounted = unmountAtom(store, a)
    aMounted?.t.delete(atom)
  }
}
```

---

### 6. 异步 Atom 与 Continuable Promise

#### 6.1 Store 层面

异步 atom 的 read 函数返回 Promise 时，store 直接将 Promise 作为值存储在 `atomState.v` 中：

```ts
setAtomStateValueOrPromise(store, atom, valueOrPromise)
if (isPromiseLike(valueOrPromise)) {
  // 注册 abort handler：当值被替换时中断旧 Promise
  registerAbortHandler(store, valueOrPromise, () => controller?.abort())
  // Promise settle 后修剪依赖并同步挂载
  const settle = () => { pruneDependencies(); mountDependenciesIfAsync() }
  valueOrPromise.then(settle, settle)
}
```

`atomState.p`（pending set）的作用：当 atom A 的 read 返回 Promise，且 A 依赖 atom B，则 A 会被加入 B 的 `p` 集合。当 B 变化时，即使 A 未挂载，A 也会被重新计算（因为 `getMountedOrPendingDependents` 会查看 `p`）。这确保了 promise 链不会断裂。

#### 6.2 React 层面 — createContinuablePromise

`src/react/useAtomValue.ts:60-102`

这是 Jotai 异步处理最精妙的部分。问题是：当一个异步 atom 的值从 Promise A 变为 Promise B（比如依赖变了触发重算），React 不应该 unmount 再 remount Suspense 子树，而应该**继续等待新 Promise**。

```ts
const createContinuablePromise = (store, promise, getValue) => {
  let continuablePromise = continuablePromiseMap.get(promise)
  if (!continuablePromise) {
    continuablePromise = new Promise((resolve, reject) => {
      let curr = promise
      const onFulfilled = (me) => (v) => { if (curr === me) resolve(v) }
      const onRejected = (me) => (e) => { if (curr === me) reject(e) }

      const onAbort = () => {
        const nextValue = getValue()         // 获取新值
        if (isPromiseLike(nextValue)) {
          continuablePromiseMap.set(nextValue, continuablePromise!)  // 关键：复用同一个 Promise
          curr = nextValue
          nextValue.then(onFulfilled(nextValue), onRejected(nextValue))
          registerAbortHandler(store, nextValue, onAbort)  // 链式注册
        } else {
          resolve(nextValue)                 // 已同步解决
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

核心思想：创建一个**包装 Promise**，当原始 Promise 被 abort 时（值被替换），不 reject，而是自动"续接"到新 Promise。对 React 来说始终是同一个 Promise 对象，Suspense 不会重置。

#### 6.3 React.use shim

```ts
const use = React.use || ((promise) => {
  if (promise.status === 'pending') throw promise       // Suspense
  else if (promise.status === 'fulfilled') return promise.value
  else if (promise.status === 'rejected') throw promise.reason
  else { attachPromiseStatus(promise); throw promise }  // 首次：标记并 throw
})
```

为不支持 `React.use` 的旧版本提供 shim，通过在 Promise 上挂载 `status/value/reason` 属性实现。

---

### 7. React 绑定层 — useAtomValue 详解

`src/react/useAtomValue.ts:119-183`

```ts
export function useAtomValue(atom, options?) {
  const store = useStore(options)

  // ① useReducer 作为外部存储同步器
  const [[valueFromReducer, storeFromReducer, atomFromReducer], rerender] =
    useReducer(
      (prev) => {
        const nextValue = store.get(atom)
        if (Object.is(prev[0], nextValue) && prev[1] === store && prev[2] === atom) {
          return prev  // ← bail-out：返回同一引用，React 不会重渲染
        }
        return [nextValue, store, atom]
      },
      undefined,
      () => [store.get(atom), store, atom],
    )

  // ② 检测 store 或 atom 引用变化（在 render 中同步处理）
  let value = valueFromReducer
  if (storeFromReducer !== store || atomFromReducer !== atom) {
    rerender()                // 标记需要重渲染
    value = store.get(atom)   // 立即使用新值渲染
  }

  // ③ 订阅 store 变更
  useEffect(() => {
    const unsub = store.sub(atom, () => {
      if (typeof delay === 'number') {
        setTimeout(rerender, delay)  // 延迟重渲染（可选）
        return
      }
      rerender()
    })
    rerender()  // 订阅后立即同步一次（防止 effect 期间有变更丢失）
    return unsub
  }, [store, atom, delay, promiseStatus])

  // ④ Promise 处理
  if (isPromiseLike(value)) {
    const promise = createContinuablePromise(store, value, () => store.get(atom))
    return use(promise)  // 触发 Suspense
  }
  return value
}
```

**为什么用 `useReducer` 而不是 `useState`？**

`useReducer` 的 reducer 在 React 的调度器中运行，如果返回同一引用（`prev`），React 会 bail-out（跳过渲染）。这比 `useState` + `Object.is` 检查更自然，也避免了在 concurrent mode 下的 tearing 问题。

**`delay` 选项的作用**：当异步 atom 频繁更新时，可以设置 delay 来合并重渲染，减少 UI 抖动。

**`useSetAtom` 的读写分离**：

```ts
export function useSetAtom(atom, options?) {
  const store = useStore(options)
  return useCallback((...args) => store.set(atom, ...args), [store, atom])
}
```

只返回 setter，**不订阅值变更**。这意味着调用 `set` 的组件不会因为值变化而重渲染。这是性能优化的关键——写入者和读取者解耦。

---

### 8. Provider 与 Store 隔离

```ts
// Provider 为子树创建独立的 Store
export function Provider({ children, store }) {
  const storeRef = useRef(null)
  if (store) {
    // 使用外部传入的 store
    return createElement(StoreContext.Provider, { value: store }, children)
  }
  if (storeRef.current === null) {
    storeRef.current = createStore()  // 惰性创建
  }
  return createElement(StoreContext.Provider, { value: storeRef.current }, children)
}

// useStore 的查找顺序：options.store > Context > defaultStore
export function useStore(options?) {
  const store = useContext(StoreContext)
  return options?.store || store || getDefaultStore()
}
```

**无 Provider 模式**：不使用 Provider 时，所有组件共享 `getDefaultStore()` 返回的全局单例。
**多 Provider 模式**：同一个 atom 对象在不同 Provider 下有独立的状态（因为 atom 只是 key，值存储在 store 的 WeakMap 中）。

---

### 9. StoreHooks — 可观测的实验性 API

```ts
type StoreHooks = {
  i?: StoreHookForAtoms  // init：atom state 首次创建
  r?: StoreHookForAtoms  // read：atom 被读取
  c?: StoreHookForAtoms  // change：atom 值变化
  m?: StoreHookForAtoms  // mount：atom 被挂载
  u?: StoreHookForAtoms  // unmount：atom 被卸载
  f?: StoreHook          // flush：回调被刷新
}
```

每个 hook 支持两种注册方式：
- `hook.add(atom, callback)` — 监听特定 atom
- `hook.add(undefined, callback)` — 监听所有 atom

这为 devtools（如 jotai-devtools）提供了完整的可观测性，无需修改核心代码。

---

### 10. 内存管理与 GC

Jotai 的内存管理依赖两个关键设计：

1. **WeakMap 存储**：`atomStateMap` 和 `mountedMap` 都是 WeakMap，当 atom 对象被 GC 时，对应的状态也会被自动回收
2. **卸载级联**：当最后一个 listener 被移除时，atom 及其不再需要的依赖会被递归卸载，从 `mountedMap` 中删除
3. **`atomState.p`（pending set）的清理**：Promise resolve/reject 后自动从依赖的 `p` 集合中移除

潜在的内存泄漏点（代码注释中也提到）：`atomState.p` 在 Promise 未 settle 之前会保持引用。如果一个 async atom 永远不 resolve，它的依赖的 `p` 集合不会被清理。

---

### 11. 设计总结与对比

| 维度 | Jotai 的选择 | 原因 |
|------|------------|------|
| 状态粒度 | 原子级（每个 atom 独立） | 精确重渲染，无 selector |
| 依赖追踪 | 运行时自动收集（getter 劫持） | 零样板代码，条件依赖自动修剪 |
| 变更检测 | Epoch 编号 + Object.is | O(1) 对比，无深比较开销 |
| 传播策略 | 推拉混合 | 已挂载 atom 推送（快），未挂载 atom 拉取（省） |
| 重算优化 | 拓扑排序 + 短路 | 最小化重算范围 |
| 批量更新 | isSync 标志 + do-while flush | 同步写入自动批量，异步也正确处理 |
| 内存模型 | WeakMap + 引用计数卸载 | 自动 GC，无内存泄漏 |
| 可扩展性 | BuildingBlocks 替换 + StoreHooks | devtools 和中间件无侵入接入 |
| 异步处理 | Continuable Promise + AbortController | Suspense 友好，自动取消过期请求 |
| React 集成 | useReducer bail-out + useEffect sub | concurrent mode 安全，精确触发 |
