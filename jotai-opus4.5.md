# Jotai 实现原理深度解析

## 一、整体架构

Jotai 的架构分为两层：

- **vanilla 层**（`src/vanilla/`）：框架无关的核心实现，包含 atom 定义、store 状态管理、依赖追踪、异步处理等
- **react 层**（`src/react/`）：基于 vanilla 层的 React 绑定，包含 Provider、useAtom、useAtomValue、useSetAtom 等

这种分层设计意味着 Jotai 的核心可以脱离 React 独立使用。

```
src/
├── vanilla/
│   ├── atom.ts            # atom 创建函数
│   ├── store.ts           # store 公共接口
│   ├── internals.ts       # store 核心实现（1000+ 行）
│   ├── typeUtils.ts       # TypeScript 类型工具
│   └── utils/             # 工具函数（atomWithReducer, selectAtom, splitAtom 等）
├── react/
│   ├── Provider.ts        # React Context Provider
│   ├── useAtomValue.ts    # 读取 atom 值的 hook
│   ├── useSetAtom.ts      # 设置 atom 值的 hook
│   ├── useAtom.ts         # 读写合一的 hook
│   └── utils/             # React 工具函数
└── babel/                 # Babel 插件（debug label、React Refresh）
```

## 二、Atom：状态的最小单元

### 2.1 atom() 函数

Atom 本质上是一个**配置对象**，不持有任何实际状态。真正的状态存储在 store 中。

```typescript
let keyCount = 0

export function atom<Value>(read, write?) {
  const key = `atom${++keyCount}`  // 全局自增 key，保证唯一性
  const config = {
    toString: () => key,           // 用于调试
  }

  if (typeof read === 'function') {
    // 派生 atom：read 是一个函数
    config.read = read
  } else {
    // 原始 atom：read 是初始值
    config.init = read
    config.read = defaultRead    // (get) => get(thisAtom)
    config.write = defaultWrite  // (get, set, arg) => set(thisAtom, typeof arg === 'function' ? arg(get(thisAtom)) : arg)
  }

  if (write) {
    config.write = write
  }
  return config
}
```

### 2.2 三种 Atom 类型

```typescript
// 1. 原始 atom：可读可写，持有初始值
const countAtom = atom(0)

// 2. 只读派生 atom：通过 getter 自动追踪依赖
const doubledAtom = atom((get) => get(countAtom) * 2)

// 3. 可写派生 atom：同时定义读取和写入逻辑
const incrementAtom = atom(
  (get) => get(countAtom),
  (get, set, delta: number) => set(countAtom, get(countAtom) + delta)
)
```

核心思想：**atom 只是描述"如何获取值"和"如何更新值"的配方，本身不存储状态**。

## 三、Store：状态管理引擎

Store 是 Jotai 的核心引擎，负责：
- 存储所有 atom 的实际状态
- 维护依赖图
- 处理状态变更和通知

### 3.1 核心数据结构

```typescript
// AtomState：每个 atom 在 store 中的状态
type AtomState<Value> = {
  d: Map<AnyAtom, EpochNumber>  // 依赖映射：依赖的 atom → 依赖时的版本号
  p: Set<AnyAtom>               // 待处理的 promise 集合
  n: EpochNumber                // 当前版本号（epoch number）
  v?: Value                     // 当前值
  e?: AnyError                  // 错误（如有）
}

// Mounted：已挂载 atom 的元信息
type Mounted = {
  l: Set<() => void>    // 监听器集合
  d: Set<AnyAtom>       // 当前依赖集合
  t: Set<AnyAtom>       // 反向依赖（谁依赖了我）
  u?: () => void        // 卸载回调
}
```

### 3.2 Epoch Number（版本号机制）

这是 Jotai 的关键优化。每个 atom 都有一个版本号 `n`：

- 当 atom 的值发生变化时，版本号递增
- 依赖记录中保存的是**依赖时的版本号**
- 检查依赖是否过期时，只需比较版本号，而不需要对值做深比较

```typescript
// 检查缓存是否有效
for (const [dep, depEpoch] of atomState.d) {
  if (readAtomState(store, dep).n !== depEpoch) {
    // 依赖的版本号变了 → 需要重新计算
    hasChangedDeps = true
    break
  }
}
```

### 3.3 Building Blocks 架构

Store 内部采用"构建块"（Building Blocks）模式，将功能分解为 25 个模块化组件：

| 编号 | 类别 | 说明 |
|------|------|------|
| 0-6 | 状态 Map | atomStateMap、mountedMap、changedAtoms 等 |
| 7-10 | 拦截器 | atomRead、atomWrite、atomOnInit、atomOnMount |
| 11-20 | 核心函数 | ensureAtomState、readAtomState、writeAtomState 等 |
| 21-23 | 公共 API | store.get、store.set、store.sub |
| 24 | 扩展钩子 | 自定义增强器 |

这种设计允许中间件式的扩展和自定义。

## 四、依赖追踪：动态依赖图

### 4.1 自动依赖收集

Jotai 不需要预先声明依赖关系。依赖是在 atom 的 `read` 函数执行过程中**动态收集**的：

```typescript
const derivedAtom = atom((get) => {
  const a = get(atomA)  // 自动记录对 atomA 的依赖
  if (a > 0) {
    return get(atomB)   // 条件性依赖：只在 a > 0 时依赖 atomB
  }
  return 0
})
```

实现机制：

```typescript
function readAtomState(store, atom) {
  const dependencies = new Map()

  // getter 函数：每次调用都记录依赖
  const getter = (depAtom) => {
    const depState = readAtomState(store, depAtom)  // 递归读取
    dependencies.set(depAtom, depState.n)           // 记录依赖及其版本号
    return depState.v
  }

  // 执行 atom 的 read 函数
  const value = atom.read(getter, options)

  // 保存依赖关系
  atomState.d = dependencies
}
```

### 4.2 依赖图是双向的

- **正向**：`AtomState.d` — 我依赖了谁
- **反向**：`Mounted.t` — 谁依赖了我

反向依赖用于在值变更时快速找到所有需要重新计算的下游 atom。

## 五、状态更新与重计算

### 5.1 写入流程

```
store.set(atom, newValue)
  ↓
writeAtomState(store, atom, newValue)
  ↓
atom.write(getter, setter, newValue)
  ↓
setter 可能更新一个或多个 atom
  ↓
标记变更的 atom（changedAtoms）
  ↓
recomputeInvalidatedAtoms()  ← 拓扑排序重计算
  ↓
flushCallbacks()  ← 通知所有监听器
```

### 5.2 拓扑排序重计算

当一个 atom 的值发生变化时，Jotai 使用**拓扑排序**来确保依赖图中的 atom 按正确顺序重新计算：

```
Step 1: DFS 遍历，构建逆拓扑序
  从 changedAtoms 出发
  沿反向依赖（Mounted.t）向上遍历
  用 visiting/visited 标记避免循环
  结果存入 topSortedReversed

Step 2: 逆序遍历，按依赖顺序重计算
  for i = topSortedReversed.length - 1 → 0:
    检查该 atom 的依赖是否在 changedAtoms 中
    如果是 → 重新读取（readAtomState）
    如果不是 → 跳过（依赖未变，无需重算）
```

这确保了：
- 底层依赖先于上层依赖被计算
- 依赖未变的 atom 不会被无谓重算
- 更新过程是**批量、高效**的

### 5.3 通知机制

```typescript
function flushCallbacks(store) {
  do {
    // 1. 收集所有已变更 atom 的监听器
    changedAtoms.forEach(atom => {
      mountedMap.get(atom)?.l.forEach(listener => callbacks.add(listener))
    })

    // 2. 收集挂载/卸载回调
    // 3. 清空追踪集合
    changedAtoms.clear()

    // 4. 执行所有回调
    callbacks.forEach(cb => cb())

    // 5. 如果回调触发了新的变更，继续循环
  } while (changedAtoms.size > 0)
}
```

## 六、挂载与卸载生命周期

### 6.1 挂载（Mount）

当组件通过 `store.sub(atom, listener)` 订阅一个 atom 时触发挂载：

```
mountAtom(atom)
  ↓
readAtomState(atom)             # 确保有值
  ↓
递归挂载所有依赖 → mountAtom(dep)
  ↓
创建 Mounted 结构 { l, d, t }
  ↓
调用 atom.onMount(setSelf)       # 生命周期回调
  ↓
返回 unsubscribe 函数
```

关键点：**只有被挂载的 atom 才会触发 React 重渲染**。未被订阅的 atom 即使值变了也不会通知任何人。

### 6.2 卸载（Unmount）

当最后一个监听器被移除时触发卸载：

```
unmountAtom(atom)
  ↓
检查是否还有监听器或被其他已挂载 atom 依赖
  ↓
如果没有 → 调用 onUnmount 回调
  ↓
从 mountedMap 中删除
  ↓
递归卸载不再需要的依赖
```

这保证了**不再使用的 atom 会被正确清理**。

## 七、React 集成层

### 7.1 Provider

```typescript
export function Provider({ children, store }) {
  const storeRef = useRef(null)
  if (!store) {
    if (storeRef.current === null) {
      storeRef.current = createStore()  // 每个 Provider 创建独立的 store
    }
    store = storeRef.current
  }
  return <StoreContext.Provider value={store}>{children}</StoreContext.Provider>
}
```

每个 Provider 拥有独立的 store 实例，实现了**状态隔离**。

### 7.2 useAtomValue

```typescript
export function useAtomValue(atom, options) {
  const store = useStore(options)

  // 用 useReducer 触发重渲染
  const [value, rerender] = useReducer(...)

  useEffect(() => {
    // 订阅 atom，值变更时触发重渲染
    const unsub = store.sub(atom, () => {
      rerender()
    })
    rerender()  // 初始同步
    return unsub
  }, [store, atom])

  // 支持异步 atom：返回 promise 时使用 React.use()
  if (isPromiseLike(value)) {
    return use(createContinuablePromise(value))
  }
  return value
}
```

### 7.3 useSetAtom

```typescript
export function useSetAtom(atom, options) {
  const store = useStore(options)
  return useCallback(
    (...args) => store.set(atom, ...args),
    [store, atom]
  )
}
```

### 7.4 useAtom

```typescript
export function useAtom(atom, options) {
  return [useAtomValue(atom, options), useSetAtom(atom, options)]
}
```

## 八、异步处理

### 8.1 异步 Atom

```typescript
const asyncAtom = atom(async (get) => {
  const id = get(idAtom)
  const response = await fetch(`/api/data/${id}`)
  return response.json()
})
```

### 8.2 Promise 状态追踪

Jotai 使用 WeakMap 追踪 promise 的状态：

```typescript
const promiseStateMap = new WeakMap<PromiseLike, [pending: boolean, abortHandlers: Set]>()
```

- 当 promise 处于 pending 状态时，依赖它的 atom 会被记录在 `AtomState.p` 中
- promise 完成后自动清理
- 支持 AbortController，可在依赖变更时中止旧请求

### 8.3 Continuable Promise

为了配合 React 的 `use()` API 和 Suspense：

```typescript
const createContinuablePromise = (promise, getValue) => {
  // 当 promise 被 abort 时（依赖变更），
  // 自动切换到新的 promise，而不是创建新的 Suspense 边界
  const onAbort = () => {
    const nextValue = getValue()
    if (isPromiseLike(nextValue)) {
      curr = nextValue  // 继续等待新 promise
    } else {
      resolve(nextValue)  // 直接 resolve
    }
  }
}
```

这避免了依赖变更时的不必要 Suspense 闪烁。

## 九、内存管理

### 9.1 WeakMap 的大量使用

Jotai 在多处使用 WeakMap：

- `atomStateMap: WeakMap<Atom, AtomState>` — atom 状态存储
- `mountedMap: WeakMap<Atom, Mounted>` — 挂载信息
- `promiseStateMap: WeakMap<Promise, State>` — promise 状态
- `continuablePromiseMap: WeakMap<Promise, Promise>` — promise 缓存

WeakMap 的关键优势：**当 atom 对象不再被引用时，相关状态会被自动垃圾回收**。

### 9.2 无全局注册表

Atom 不需要在全局注册表中注册。它们只是普通的 JavaScript 对象，遵循正常的 GC 规则。这意味着：

- 动态创建的 atom 在不再使用后可以被回收
- 不存在内存泄漏的隐患
- 不需要手动清理

## 十、工具函数设计模式

### 10.1 atomWithReducer

```typescript
export function atomWithReducer(initialValue, reducer) {
  return atom(initialValue, function (get, set, action) {
    set(this, reducer(get(this), action))  // this 指向当前 atom
  })
}
```

### 10.2 selectAtom — 多级 WeakMap 缓存

```typescript
// 使用三级 WeakMap 做 memoization
const cache1 = new WeakMap()
const memo3 = (create, dep1, dep2, dep3) => {
  const cache2 = getCached(() => new WeakMap(), cache1, dep1)
  const cache3 = getCached(() => new WeakMap(), cache2, dep2)
  return getCached(create, cache3, dep3)
}
```

保证 `selectAtom(baseAtom, selector, equalityFn)` 在相同参数下返回同一个派生 atom 实例。

### 10.3 splitAtom

将一个数组 atom 拆分为多个独立的 item atom，通过 key 追踪实现稳定的身份标识，避免数组变动时不必要的重渲染。

### 10.4 atomWithStorage

```typescript
export function atomWithStorage(key, initialValue, storage) {
  const baseAtom = atom(initialValue)
  baseAtom.onMount = (setSelf) => {
    // 挂载时从 storage 同步值
    const value = storage.getItem(key)
    if (value !== undefined) setSelf(value)
    // 可选：监听 storage 变更事件
    return storage.subscribe?.(key, setSelf)
  }
  // 写入时同步到 storage
  return atom(
    (get) => get(baseAtom),
    (get, set, value) => {
      set(baseAtom, value)
      storage.setItem(key, value)
    }
  )
}
```

## 十一、关键设计决策总结

| 设计决策 | 选择 | 原因 |
|---------|------|------|
| 状态存储位置 | Store（非 atom 自身） | 支持多 store、状态隔离、可测试性 |
| 依赖声明方式 | 动态收集（getter 追踪） | 灵活、条件性依赖、无需手动维护 |
| 缓存失效策略 | Epoch number 版本号 | 高效、避免深比较、O(1) 检查 |
| 更新顺序 | 拓扑排序 | 保证依赖先于被依赖者更新 |
| 内存管理 | WeakMap | 自动 GC、无全局注册表 |
| 框架绑定 | 分层架构 | 核心逻辑可复用、可独立测试 |
| 异步处理 | Promise + AbortController | 原生 async/await、Suspense 兼容 |
| 扩展机制 | Building Blocks + Store Hooks | 中间件式扩展、不侵入核心 |

## 十二、一次完整的更新流程

以下是用户点击按钮更新 `countAtom`，导致 `doubledAtom` 重新计算并触发 UI 更新的完整流程：

```
用户点击 → setCount(prev => prev + 1)
  ↓
store.set(countAtom, updater)
  ↓
writeAtomState: 调用 countAtom.write(get, set, updater)
  ↓
defaultWrite: set(countAtom, updater(get(countAtom)))
  → countAtom 的值从 0 变为 1
  → countAtom.n（版本号）递增
  → countAtom 加入 changedAtoms
  ↓
recomputeInvalidatedAtoms:
  1. 从 changedAtoms 出发，DFS 遍历反向依赖
     → 发现 doubledAtom 依赖了 countAtom
  2. 拓扑排序后逆序重计算
     → readAtomState(doubledAtom)
     → 执行 (get) => get(countAtom) * 2
     → 返回 2
     → doubledAtom.n 递增
     → doubledAtom 加入 changedAtoms
  ↓
flushCallbacks:
  → 遍历 changedAtoms
  → 收集 countAtom 和 doubledAtom 的所有 listener
  → 执行 listener（即 useAtomValue 中的 rerender）
  ↓
React 重渲染使用了这些 atom 的组件
```

## 总结

Jotai 的实现体现了几个核心理念：

1. **原子化**：状态被拆分为最小的独立单元（atom），组合而非集中
2. **惰性计算**：只有被订阅的 atom 才会参与计算和更新
3. **自动依赖追踪**：开发者无需手动声明依赖，getter 调用即声明
4. **精确更新**：拓扑排序 + 版本号机制确保最小化重计算范围
5. **零配置**：atom 即用即创建，无需 Provider（有默认 store）、无需 action type、无需 reducer 模板

整体代码量精简（核心实现约 1000 行），但通过 epoch number、拓扑排序、WeakMap 等精巧设计实现了高效的响应式状态管理。
