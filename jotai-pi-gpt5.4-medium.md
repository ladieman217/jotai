# Jotai repo 实现原理

这个 repo 是 **Jotai**，一个很薄的 React 状态管理库。

一句话概括：

> **atom 只是状态的配置对象，真正的状态存在 store 里；React 层负责把 store 和组件订阅连接起来。**

---

## 1. 整体分层

关键目录：

- `src/vanilla/*`：核心状态引擎，不依赖 React
- `src/react/*`：React 绑定层
- `src/vanilla/utils/*`：基于 atom 组合出来的高级工具
- `src/index.ts`：统一导出

入口非常薄：

- `src/index.ts`
- `src/vanilla.ts`
- `src/react.ts`

设计思路很清晰：

1. 先实现通用 store / atom runtime
2. 再做 React hook 适配
3. 最后用 atom 组合 atom 做各种 util

---

## 2. atom 本质是什么

看 `src/vanilla/atom.ts`。

Jotai 的 `atom()` **不会直接创建状态**，而是返回一个配置对象：

- primitive atom：带 `init`
- derived atom：带 `read`
- writable derived atom：带 `write`
- 可选 `onMount`

常见字段：

- `init`：初始值
- `read(get, options)`：如何计算当前值
- `write(get, set, ...args)`：如何更新
- `onMount(setAtom)`：挂载时副作用
- `toString()`：调试 key

所以：

- `atom(0)` 不是“状态实体”
- 它只是“这个 atom 默认值是 0”的声明

真正的值放在 store 里。

---

## 3. 为什么不需要字符串 key

Jotai 直接把 **atom 对象本身** 当标识。

store 里核心结构是：

- `WeakMap<Atom, AtomState>`

也就是：

- key = atom 对象
- value = 这个 atom 在当前 store 中的运行时状态

好处：

1. 不需要全局命名
2. 天然支持多个 store
3. `WeakMap` 对 GC 友好，atom 没有引用后状态也可回收

---

## 4. store 才是核心

看：

- `src/vanilla/store.ts`
- `src/vanilla/internals.ts`

公开 store API 很小：

- `get(atom)`
- `set(atom, ...args)`
- `sub(atom, listener)`

`createStore()` 最终调用 `INTERNAL_buildStoreRev2()` 构建真正的运行时。

### 默认 store

如果你不写 `<Provider>`，Jotai 会使用一个全局默认 store，也就是 provider-less mode。

但 SSR 场景不推荐这样，因为默认 store 会跨请求共享。

---

## 5. store 内部维护了什么

`src/vanilla/internals.ts` 是核心文件。

### 5.1 `atomStateMap`

```ts
WeakMap<Atom, AtomState>
```

`AtomState` 主要字段：

- `d: Map<依赖atom, 依赖的epoch>`
- `p: Set<pending promise 的依赖方>`
- `n: epoch number`
- `m: 上次校验时的 store epoch`
- `v: value`
- `e: error`

可以理解成：

- `v/e`：当前值或错误
- `d`：我依赖谁
- `n`：我的版本号
- `m`：缓存是否有效
- `p`：异步场景的额外追踪

### 5.2 `mountedMap`

```ts
WeakMap<Atom, Mounted>
```

只给“已挂载”的 atom 维护额外信息：

- `l`：订阅 listener 集合
- `d`：这个 atom 当前挂载着的依赖
- `t`：依赖这个 atom 的下游 atom
- `u`：卸载时 cleanup

### 5.3 `changedAtoms` / `invalidatedAtoms`

用于增量更新：

- `changedAtoms`：本轮真实发生变化的 atom
- `invalidatedAtoms`：缓存失效、需要重算的 atom

---

## 6. 读 atom：依赖收集 + 缓存 + 增量计算

核心函数：

- `BUILDING_BLOCK_readAtomState`

### 6.1 primitive atom

如果 atom 有 `init`，第一次读时会把 `init` 写进状态。

### 6.2 derived atom

执行某个 atom 的 `read(get)` 时：

- 每次 `get(otherAtom)`
- Jotai 先读取 `otherAtom`
- 再把 `otherAtom` 记录进当前 atom 的依赖表 `d`
- 同时记录依赖当时的 epoch

读完后，Jotai 就知道：

> 当前 atom 依赖哪些 atom，以及这些依赖当时的版本号。

### 6.3 缓存判断

Jotai 不会每次都重算 derived atom。它会先检查：

1. 当前 atom 是否已有状态
2. 是否 mounted
3. 是否被 invalidated
4. 依赖 epoch 是否变化

如果依赖没变，就直接复用缓存值。

---

## 7. 写 atom：递归 set + 失效传播

核心函数：

- `BUILDING_BLOCK_writeAtomState`
- `BUILDING_BLOCK_invalidateDependents`

### 7.1 primitive atom 写入

`set(atom, newValue)` 最终会：

1. 更新 `atomState.v`
2. 值有变化则 `epoch n++`
3. 把 atom 放进 `changedAtoms`
4. 递归标记下游 dependents 为 invalidated

### 7.2 writable derived atom 写入

writable derived atom 的 `write(get, set, ...args)` 可以：

- 读别的 atom
- 写别的 atom
- 一次写多个 atom

Jotai 的 `set` 是递归的：

- 如果写的是当前目标 atom，就直接更新
- 否则递归进入目标 atom 的 `writeAtomState`

所以 derived atom 很像 action / command。

---

## 8. 为什么更新后不会全量重算

Jotai 分两步处理更新：

### 第一步：失效传播

`invalidateDependents(store, atom)`

从被修改 atom 出发，把下游依赖链标记为 invalidated。

### 第二步：拓扑顺序重算

`recomputeInvalidatedAtoms(store)`

它会沿依赖图 DFS，得到一个近似拓扑顺序，然后从底向上重算受影响 atom。

这意味着：

- 只重算依赖链上的 atom
- 顺序正确
- 不相关分支不会受影响

---

## 9. mounted / unmounted 机制

核心函数：

- `mountAtom`
- `mountDependencies`
- `unmountAtom`
- `store.sub`

### 9.1 什么时候 mount

React 组件调用 `useAtomValue/useAtom` 后，最终会走：

- `store.sub(atom, listener)`

这会：

1. `mountAtom(atom)`
2. 递归 mount 它的依赖
3. 注册 listener

### 9.2 什么时候 unmount

组件卸载时：

1. 删除 listener
2. `unmountAtom(atom)`
3. 如果既没有监听者，也不是别人的依赖，就继续递归卸载依赖

### 9.3 `onMount`

如果 atom 定义了 `onMount`，Jotai 会在挂载后调用它，并保存 cleanup。

常用于：

- 初始化订阅
- 连接外部数据源
- 卸载清理

---

## 10. React 层其实很薄

主要文件：

- `src/react/Provider.ts`
- `src/react/useAtom.ts`
- `src/react/useAtomValue.ts`
- `src/react/useSetAtom.ts`

### 10.1 `Provider`

作用就是：

- 建一个 React Context
- 往里面塞 store
- 如果没传 store，就内部 `createStore()`

所以 Provider 只是 store 注入器。

### 10.2 `useSetAtom`

本质上就是：

```ts
const store = useStore()
return useCallback((...args) => store.set(atom, ...args), [store, atom])
```

### 10.3 `useAtom`

本质上：

- `useAtomValue(atom)`
- `useSetAtom(atom)`

拼成 tuple 返回。

### 10.4 `useAtomValue`

这是 React 层最复杂的部分，负责：

- 订阅 store
- 触发 rerender
- 处理 Promise / Suspense
- React 版本兼容

它内部通过 `useReducer` 维护上一轮的 value/store/atom。

---

## 11. Async atom 的实现原理

Jotai v2 的理念是：

> **async atom 没有特殊类型，它只是“值恰好是 Promise 的 atom”。**

也就是：

- `read` 返回普通值 → 同步 atom
- `read` 返回 Promise → 异步 atom

### 11.1 store 层

在 `setAtomStateValueOrPromise` 中，Promise 也会被直接放到 `atomState.v`。

store 不会强行 await，只会维护：

- 当前值是 Promise
- 旧 Promise 被替换时需要 abort

### 11.2 React 层接入 Suspense

`useAtomValue.ts` 里，如果拿到 Promise：

- 包装成 `continuablePromise`
- 再交给 `React.use(...)` 或 shim

这样就能接入 Suspense。

### 11.3 continuablePromise 的意义

场景：

1. atom 当前返回 `P1`
2. 依赖变化后又返回 `P2`
3. 组件不能一直傻等旧的 `P1`

所以 Jotai 会把等待过程包装成“可续接 promise”：

- 当前在等 `P1`
- 如果 `P1` 被替换/abort
- 自动切换去等 `P2`

这样异步 atom 的行为会更自然。

---

## 12. 为什么能减少无意义重渲染

Jotai 的粒度是 **atom 级别**。

例如：

- `nameAtom`
- `ageAtom`

如果某组件只读 `nameAtom`，那 `ageAtom` 更新时它不会 rerender。

根本原因有两层：

1. 数据层：依赖图精确到 atom
2. 订阅层：组件只订阅自己读取的 atom

这和“全局大 store + selector”思路不同。

---

## 13. utils 的实现哲学

Jotai 很多高级功能都不是硬塞进核心 runtime，而是通过 **atom 组合 atom** 实现。

例如：

- `atomWithDefault`
- `atomWithReset`
- `selectAtom`
- `splitAtom`
- `atomWithStorage`
- `loadable`
- `unwrap`

### 13.1 `atomWithDefault`

看 `src/vanilla/utils/atomWithDefault.ts`。

思路：

- 内部建一个 `overwrittenAtom`
- 如果用户覆盖过，就返回覆盖值
- 否则走默认 `getDefault`

### 13.2 `selectAtom`

看 `src/vanilla/utils/selectAtom.ts`。

思路：

- 基于原 atom 派生一个新 atom
- 在 selector 结果上做 `equalityFn`
- 如果 slice 没变，就复用旧值

### 13.3 `splitAtom`

看 `src/vanilla/utils/splitAtom.ts`。

思路：

- 把数组 atom 拆成多个 item atom
- 每个 item atom 负责映射到原数组某一项
- 写 item atom 时再回写原数组 atom

这说明 Jotai 的抽象足够强，很多高级功能都能通过组合表达出来。

---

## 14. 设计关键词

我会用这几个词总结这个 repo：

### 1) config / runtime 分离

- atom 是配置
- store 是运行时

### 2) 弱引用存储

- `WeakMap<atom, state>`
- 对 GC 友好

### 3) 依赖图增量更新

- 记录依赖关系
- 用版本号判断是否失效
- 只重算必要节点

### 4) React 只是适配层

- 核心不依赖 React
- React hook 只负责订阅和 Suspense 对接

### 5) 高级能力靠组合

- utils 尽量通过 atom 组合实现
- 不让核心过度膨胀

---

## 15. 一个直观类比

可以把 Jotai 类比成 Excel：

- **atom**：单元格定义
- **derived atom**：公式单元格
- **store**：整张表当前计算状态
- **useAtom**：把某个单元格绑定到 React 组件
- **recomputeInvalidatedAtoms**：公式重算引擎

所以它本质上像一个小型的**响应式依赖图执行器**，只不过输出目标是 React。

---

## 16. 最值得读的源码

如果你想抓主线，建议重点看：

1. `src/vanilla/atom.ts`
   - atom 配置对象怎么定义

2. `src/vanilla/internals.ts`
   - `readAtomState`
   - `writeAtomState`
   - `invalidateDependents`
   - `recomputeInvalidatedAtoms`
   - `mountAtom`
   - `unmountAtom`

3. `src/react/useAtomValue.ts`
   - React 订阅与 Suspense 是怎么接起来的

4. `src/vanilla/utils/atomWithDefault.ts`
5. `src/vanilla/utils/selectAtom.ts`
   - 看高级能力怎么通过 atom 组合出来

---

## 17. 一条最重要的调用链：`useAtom(countAtom)` 到组件重渲染

如果把整个 repo 串成一条线，最关键的调用链大概是这样：

```txt
组件 render
  -> useAtom(atom)
    -> useAtomValue(atom)
      -> store.get(atom)
        -> readAtomState(atom)
          -> 如有需要执行 atom.read(get)
          -> 收集依赖、返回值
    -> useSetAtom(atom)
      -> 返回 store.set(atom, ...args)

组件 commit 后
  -> useEffect 里 store.sub(atom, listener)
    -> mountAtom(atom)
    -> 注册 listener

后续某次写入
  -> store.set(atom, update)
    -> writeAtomState(atom, update)
    -> changedAtoms + invalidateDependents
    -> recomputeInvalidatedAtoms()
    -> flushCallbacks()
    -> listener 执行
    -> React rerender
```

这里最值得注意的是：

1. **render 阶段负责读取**
2. **effect 阶段负责订阅**
3. **store 层负责依赖追踪和增量更新**
4. **React 只是在 listener 触发后重渲染**

也就是说，Jotai 并没有把 React 状态逻辑塞到 atom 里，而是把 React 当成一个“订阅了 store 的视图层”。

---

## 18. 一条写入调用链：`setCount((c) => c + 1)` 内部发生了什么

以 primitive atom 为例：

```ts
const countAtom = atom(0)
const [count, setCount] = useAtom(countAtom)
setCount((c) => c + 1)
```

内部可以理解为：

### 第一步：`useSetAtom` 返回 setter

`useSetAtom` 最终只是包了一层：

```ts
(...args) => store.set(atom, ...args)
```

### 第二步：进入 `store.set`

`store.set` 会调用 `writeAtomState`。

### 第三步：primitive atom 的默认 write 生效

在 `src/vanilla/atom.ts` 里，primitive atom 有默认 `write`：

- 如果传入函数，就拿旧值算新值
- 否则直接把传入值作为新值

### 第四步：真正更新 atom state

`writeAtomState` 最终会：

- 调 `setAtomStateValueOrPromise`
- 比较新旧值是否变化
- 变化则让 `atomState.n++`
- 把该 atom 放进 `changedAtoms`
- 调 `invalidateDependents` 标记下游节点失效

### 第五步：重算并通知

`store.set` 结束前会触发：

- `recomputeInvalidatedAtoms()`
- `flushCallbacks()`

然后所有订阅这个 atom 的 React 组件 listener 会收到通知，最后 rerender。

所以 Jotai 写入的核心节奏其实是：

> **写入当前 atom → 标记依赖图失效 → 按需重算 → 通知组件**

---

## 19. 一条读取调用链：derived atom 是怎么被计算出来的

假设：

```ts
const countAtom = atom(1)
const doubledAtom = atom((get) => get(countAtom) * 2)
```

当组件读取 `doubledAtom` 时：

### 第一步：进入 `readAtomState(doubledAtom)`

Jotai 先看缓存能不能直接复用：

- 这个 atom 有没有旧值
- 它是不是 mounted
- 它有没有被 invalidated
- 它依赖的 epoch 有没有变

如果都没变，就直接返回旧值。

### 第二步：执行 `doubledAtom.read(get)`

如果不能走缓存，就执行 read 函数。

### 第三步：在 `get(countAtom)` 时收集依赖

这一步是关键：

- `readAtomState(countAtom)` 会先拿到 `countAtom` 当前状态
- 再把 `countAtom` 记录到 `doubledAtom` 的依赖表 `d`
- 同时记录 `countAtom` 当前的 epoch

### 第四步：保存本次结果

read 返回后，`setAtomStateValueOrPromise` 会把新值写进 `doubledAtom` 的 atom state。

这样下次再读时，Jotai 就知道：

- `doubledAtom` 依赖 `countAtom`
- 上次计算时，`countAtom` 的版本号是 N

只要下次 `countAtom` 的版本号还是 N，就不用重算。

---

## 20. Async atom 的完整心智模型

前面说过：async atom 本质上只是“值是 Promise 的 atom”。这里再把它完整说一下。

假设：

```ts
const userIdAtom = atom(1)
const userAtom = atom(async (get, { signal }) => {
  const id = get(userIdAtom)
  const res = await fetch(`/api/users/${id}`, { signal })
  return res.json()
})
```

### 20.1 首次读取

组件 `useAtomValue(userAtom)`：

- `store.get(userAtom)`
- `readAtomState(userAtom)`
- 执行 async read
- 返回一个 Promise
- Promise 被保存到 `atomState.v`

### 20.2 React 层处理 Promise

`useAtomValue` 发现值是 Promise，会：

- 构造 `continuablePromise`
- 交给 `React.use()` 或兼容 shim
- 让组件进入 Suspense

### 20.3 Promise resolve 后

resolve 时：

- atom 的值更新为最终结果
- listener 被通知
- 组件重新渲染
- 这次读到的就不再是 Promise，而是最终值

### 20.4 如果依赖中途变化

如果等待过程中 `userIdAtom` 变化，`userAtom` 会重新计算并产出一个新的 Promise。

Jotai 不是让组件死等旧 Promise，而是：

- 给旧 Promise 注册 abort handler
- 在新 Promise 到来时切换等待目标
- continuablePromise 继续跟随最新的 Promise

这就是为什么 Jotai 的异步体验比“自己手写 useEffect + loading state”更接近声明式数据依赖。

### 20.5 `signal` 的作用

`read(get, { signal })` 中的 `signal` 是 Jotai 给 async read 的中断能力。

如果旧 Promise 已经过时：

- Jotai 会触发对应 abort handler
- 你在 fetch 中传入的 `signal` 就能取消旧请求

这能避免：

- 过时请求继续占用资源
- 旧结果晚到、覆盖新结果

---

## 21. `flushCallbacks` 为什么重要

`flushCallbacks` 是这个实现里一个容易被忽略但很关键的函数。

它负责把本轮累积的事情统一冲刷出去，包括：

- 变更 atom 的 listener
- onMount 回调
- onUnmount 回调

它的意义在于：

### 21.1 批量化通知

不是每改一个 atom 就立刻同步通知组件，而是先累计到：

- `changedAtoms`
- `mountCallbacks`
- `unmountCallbacks`

然后统一 flush。

### 21.2 支持连锁更新

`flushCallbacks` 里如果又导致新的 `changedAtoms`，Jotai 会继续：

- `recomputeInvalidatedAtoms`
- 再次 flush

直到没有新的变化为止。

所以它像一个“小事务提交循环”。

---

## 22. 为什么 Jotai 既有“未挂载状态”又有“挂载状态”

这是源码里一个容易看晕的点。

Jotai 同时维护：

- `atomStateMap`：所有被 store 见过的 atom 状态
- `mountedMap`：当前活跃订阅图

### 原因

一个 atom 即使现在没有组件订阅，它的值也仍然可能需要被读取或写入。

比如：

- 你在某个 action atom 里 `get(otherAtom)`
- 或你直接 `store.get(atom)`

这时它必须有 state，但不一定要有 mounted 信息。

### 区别

#### `atomStateMap`

偏“数据缓存层”：

- 当前值
- 错误
- 依赖表
- epoch

#### `mountedMap`

偏“活动订阅层”：

- 哪些 listener 在监听
- 哪些依赖现在需要保持挂载
- unmount 时要做哪些 cleanup

这种拆分让 Jotai 既能做缓存，又能在订阅层面做精细清理。

---

## 23. `Provider` 的真实意义：不是必须，但能隔离 store

很多初学者会以为 `Provider` 是必须的，其实不是。

Jotai 在没有 `Provider` 时，会退回到默认 store：

- `getDefaultStore()`

所以最简单的 demo 可以直接写。

但 `Provider` 很重要，因为它解决了两个问题：

### 23.1 状态隔离

同一个 atom 配置对象，在不同 store 中可以有不同值。

也就是说：

- atom 是配置
- store 才是状态容器

所以两个不同 Provider 子树，完全可以共享同一套 atom 定义，但运行出两份独立状态。

### 23.2 SSR 安全

如果用默认 store，服务端多个请求可能共享同一个 store。

而用了 Provider：

- 每次请求创建自己的 store
- 生命周期局限在该次请求
- 避免跨请求串状态

---

## 24. SSR / hydration 在这个 repo 里的思路

相关实现：

- `src/react/utils/useHydrateAtoms.ts`

它的实现思路很直接：

- 拿到 store
- 遍历传入的 `[atom, ...args]`
- 对尚未 hydrate 的 atom 调一次 `store.set(atom, ...args)`
- 用 `WeakMap<Store, WeakSet<Atom>>` 记录“某个 store 已经 hydrate 过哪些 atom”

也就是说 hydration 本质不是一套额外机制，而是：

> **把初始值通过 store.set 灌进指定 store，只是加了一层“每个 store 只 hydrate 一次”的保护。**

这也解释了为什么文档里说：

- atom 只能在同一个 store 上 hydrate 一次
- 如果强行重灌，要用 `dangerouslyForceHydrate`

---

## 25. 几个 util 如何体现“组合优于内建”

除了前面提到的 `atomWithDefault`、`selectAtom`、`splitAtom`，还有几个很能体现 Jotai 风格的 util。

### 25.1 `unwrap`

文件：`src/vanilla/utils/unwrap.ts`

作用：把 async atom 变成一个“同步可读”的 atom，并允许你提供 pending fallback。

核心思路：

- 内部缓存 Promise 的结果或错误
- Promise resolve / reject 后，触发一个私有 refresh atom 自增
- 派生 atom 重新读取时，就能同步返回：
  - 已完成值
  - fallback
  - 或抛出错误

这其实相当于：

> **把 Promise 生命周期外显成另一个 atom 驱动的同步状态机。**

### 25.2 `loadable`

文件：`src/vanilla/utils/loadable.ts`

它现在已经被标记为 deprecated，但很适合看设计思路。

本质上它就是：

- 先 `unwrap(anAtom, () => LOADING)`
- 再包一层 atom
- 返回 `{ state: 'loading' | 'hasData' | 'hasError' }`

也就是说，它并不是核心特性，而是一个“基于 unwrap 的用户态封装”。

### 25.3 `atomWithStorage`

文件：`src/vanilla/utils/atomWithStorage.ts`

思路是：

- 用一个内部 `baseAtom` 保存当前内存值
- `onMount` 时从 storage 读一次并订阅 storage 变化
- 写入时同时更新 `baseAtom` 和持久化存储

这说明 Jotai 不把“持久化状态”当特殊状态类型，而是：

> **普通 atom + 外部存储同步策略**

### 25.4 `useAtomCallback`

文件：`src/react/utils/useAtomCallback.ts`

这个 hook 的实现也很简洁：

- 在 `useMemo` 里创建一个 write-only atom
- atom 的 write 里调用用户传入 callback
- 再把这个 atom 交给 `useSetAtom`

所以“imperative 地读写 atom”也不是新增 runtime API，而是借已有写入机制包出来的。

---

## 26. Jotai 的一个关键心智模型：atom 不是 store 里的 slot，而是图上的节点

如果把 Jotai 想成“key-value 状态容器”，会越看越别扭。

更合适的理解是：

- primitive atom：图上的源节点
- derived atom：图上的计算节点
- writable derived atom：图上的动作节点
- store：这张图在某次运行时的具体状态

所以 Jotai 最像的不是 Redux，而更像：

- 响应式图执行器
- 增量求值系统
- 面向 React 的最小依赖追踪 runtime

这个视角能解释很多设计：

- 为什么 atom 本身不存值
- 为什么依赖关系是在 read 时动态收集
- 为什么只需要按需重算下游节点
- 为什么高级能力都能继续表示成 atom

---

## 27. 与 Redux / Zustand / Recoil 的实现风格差异

这里只讲实现风格，不讲谁更好。

### 对 Redux

Redux 更像：

- 单一大状态树
- dispatch action
- reducer 纯函数更新
- 组件通过 selector 取局部视图

Jotai 更像：

- 天然分片的状态节点
- 不需要先把所有状态揉成一棵树
- 依赖关系由 atom read 自动建立

### 对 Zustand

Zustand 更像：

- 一个中心 store
- 组件订阅 store 的某个 slice
- 通过 selector + equality 控制重渲染

Jotai 更偏：

- store 里不是一个大对象，而是一堆 atom state
- 依赖追踪在 atom graph 层完成，而不是只在 selector 层完成

### 对 Recoil

Recoil 跟 Jotai 在“atom / selector 图”上更接近。

但 Jotai 更轻的一点是：

- 不依赖字符串 key
- 更强调 atom 配置对象本身做 identity
- 核心 API 更小

---

## 28. 阅读这个 repo 时最容易误解的几个点

### 28.1 误解一：atom 就是 state

不是。

更准确地说：

- atom = state definition / config
- store 里的 atomState = runtime state

### 28.2 误解二：derived atom 像 memo selector

只说对了一半。

它确实像 selector，但更准确地说它是：

- 会自动追踪依赖的计算节点
- 同时还能是 writable action 节点

### 28.3 误解三：Provider 只是 React Context 包装

表面上是，实质上它决定了：

- 用哪个 store
- 状态是否隔离
- SSR 是否安全

### 28.4 误解四：async atom 只是 Suspense 语法糖

不只是。

它背后还有：

- Promise 生命周期跟踪
- abort handler
- continuablePromise
- pending dependency 跟踪

### 28.5 误解五：utils 只是语法糖

有些是语法糖，但很多 util 本身也展示了 Jotai 的核心哲学：

- 状态能力尽量由 atom 组合出来
- 而不是不断扩大内核 API 面

---

## 29. 如果你要自己实现一个迷你版 Jotai，最小需要哪些能力

可以按这个顺序做：

### 第一步：primitive atom + store + subscribe

最小版：

- `atom(init)`
- `WeakMap<atom, { value, listeners }>`
- `useAtom(atom)`

### 第二步：derived atom + 依赖追踪

再加：

- `atom(read)`
- read 时 `get(dep)` 自动收集依赖
- 某个依赖变化时递归通知下游

### 第三步：缓存与版本号

再加：

- 每个 atom state 维护 epoch
- 依赖没变时复用缓存

### 第四步：挂载与卸载

再加：

- mounted listeners
- 依赖挂载关系
- onMount / cleanup

### 第五步：异步与 Suspense

最后才是：

- Promise 值
- abort
- continuablePromise
- hydration / utils

换句话说，这个 repo 虽然文件不算少，但真正的主干并不复杂：

> **读、写、依赖追踪、失效传播、订阅通知。**

---

## 30. 最后的总结

**Jotai 的实现原理就是：**

- 用 `atom` 对象描述状态单元和依赖关系
- 用 `WeakMap` 在 `store` 中保存每个 atom 的运行时状态
- 在 read 时动态收集依赖，在 write 时传播失效
- 用 epoch 和缓存避免无意义重算
- 用 mounted 图管理订阅、生命周期和依赖挂载
- 用很薄的 React 层把 store 和组件连接起来
- 用 atom 组合 atom 的方式扩展出大量高级能力

如果只记一句话，我会建议记这个：

> **Jotai 不是“一个大 store + selector 系统”，而是“一个以 atom 为节点、按需增量求值的细粒度响应式状态图”。**

---

## 31. 源码标注版：结论分别对应到哪些文件

下面把前面的核心结论，直接映射到源码位置，方便你二刷源码时对照看。

### 31.1 `atom` 只是配置对象，不持有运行时状态

对应文件：`src/vanilla/atom.ts`

关键点：

- `atom(...)` 根据参数形态构造 config 对象
- primitive atom 会带 `init`
- derived atom 会带 `read`
- writable atom 会带 `write`
- `defaultRead` / `defaultWrite` 负责 primitive atom 的默认行为

可以重点看：

- `export function atom(...)`
- `defaultRead`
- `defaultWrite`

结论：

> atom 本身只是“定义”，不是“状态存储体”。

### 31.2 store 是真正的运行时容器

对应文件：

- `src/vanilla/store.ts`
- `src/vanilla/internals.ts`

关键点：

- `createStore()` 返回公开 API：`get / set / sub`
- `getDefaultStore()` 提供 provider-less 模式
- `INTERNAL_buildStoreRev2` 构造真正的 store runtime

可以重点看：

- `createStore`
- `getDefaultStore`
- `buildStore as INTERNAL_buildStoreRev2`

结论：

> 真正的值、依赖关系、订阅关系都存在 store 里。

### 31.3 atom 状态通过 `WeakMap<Atom, AtomState>` 保存

对应文件：`src/vanilla/internals.ts`

关键点：

- `AtomState` 类型定义了 `d / p / n / m / v / e`
- `BUILDING_BLOCK_ensureAtomState` 里首次为 atom 建立 state

可以重点看：

- `type AtomState`
- `BUILDING_BLOCK_ensureAtomState`

结论：

> Jotai 用 atom 对象自身作为 identity，用 WeakMap 存运行时状态。

### 31.4 依赖是在 read 时动态收集的

对应文件：`src/vanilla/internals.ts`

关键函数：

- `BUILDING_BLOCK_readAtomState`

重点逻辑：

- 在内部 `getter(a)` 中调用 `readAtomState(store, a)`
- 然后把 `a` 记录进 `atomState.d`
- 把依赖当时的 epoch 也一起记下

结论：

> 依赖关系不是静态声明的，而是在执行 `atom.read(get)` 时动态建立的。

### 31.5 缓存复用靠 epoch 和依赖检查

对应文件：`src/vanilla/internals.ts`

关键函数：

- `BUILDING_BLOCK_readAtomState`

重点逻辑：

- 已初始化 atom 先尝试走缓存
- 如果 mounted 且没 invalidated，可直接复用
- 否则检查依赖 `d` 中记录的 epoch 是否变化
- 未变化就复用旧值

结论：

> derived atom 不是每次 render 都会重算，而是按依赖版本号增量复用。

### 31.6 写入会触发失效传播

对应文件：`src/vanilla/internals.ts`

关键函数：

- `BUILDING_BLOCK_writeAtomState`
- `BUILDING_BLOCK_invalidateDependents`
- `BUILDING_BLOCK_setAtomStateValueOrPromise`

重点逻辑：

- `setAtomStateValueOrPromise` 更新值并推进 epoch
- `writeAtomState` 把 atom 放进 `changedAtoms`
- `invalidateDependents` 递归把下游节点标记 invalidated

结论：

> 写入不是“全量刷新”，而是“从改动点向下游传播失效”。

### 31.7 重算不是乱序的，而是按依赖图顺序完成

对应文件：`src/vanilla/internals.ts`

关键函数：

- `BUILDING_BLOCK_recomputeInvalidatedAtoms`

重点逻辑：

- 先 DFS 收集受影响 atom
- 形成一个反向拓扑顺序列表
- 再逆序重算真正受影响的节点

结论：

> Jotai 的重算顺序是依赖有序的，不是简单暴力递归。

### 31.8 mounted 图负责订阅生命周期

对应文件：`src/vanilla/internals.ts`

关键函数：

- `BUILDING_BLOCK_storeSub`
- `BUILDING_BLOCK_mountAtom`
- `BUILDING_BLOCK_mountDependencies`
- `BUILDING_BLOCK_unmountAtom`

重点逻辑：

- `store.sub` 时先 `mountAtom`
- mount 时递归挂载依赖
- unsub 时若不再被需要，则递归卸载

结论：

> Jotai 不只是维护值，还维护“当前哪些依赖链处于活跃订阅状态”。

### 31.9 `onMount` 是 atom 级生命周期扩展点

对应文件：

- `src/vanilla/atom.ts`
- `src/vanilla/internals.ts`

关键函数：

- `WritableAtom['onMount']`
- `BUILDING_BLOCK_mountAtom`
- `BUILDING_BLOCK_atomOnMount`

结论：

> atom 可以在第一次被挂载到活跃图时启动副作用，并在卸载时清理。

### 31.10 React 层只负责接 store

对应文件：

- `src/react/Provider.ts`
- `src/react/useAtom.ts`
- `src/react/useAtomValue.ts`
- `src/react/useSetAtom.ts`

关键点：

- `Provider` 提供 store context
- `useStore` 从 context 或默认 store 中取 store
- `useSetAtom` 只是把 `store.set` 包装成 callback
- `useAtom` 只是 `useAtomValue + useSetAtom`

结论：

> React 层并不是核心状态机，只是桥接层。

### 31.11 Suspense 接入主要在 `useAtomValue`

对应文件：`src/react/useAtomValue.ts`

关键点：

- 检查值是不是 Promise
- `createContinuablePromise` 跟踪 promise 替换
- `use(promise)` / shim 与 React Suspense 对接

结论：

> store 负责保存 Promise，React 层负责把 Promise 接到 Suspense 语义上。

### 31.12 hydration 本质上就是一次受控的 `store.set`

对应文件：`src/react/utils/useHydrateAtoms.ts`

关键点：

- `WeakMap<Store, WeakSet<Atom>>` 记录每个 store 已 hydrate 的 atom
- 对未 hydrate 的 atom 调用 `store.set(atom, ...args)`

结论：

> hydration 在实现上非常朴素，没有额外神秘机制。

### 31.13 util 的设计是“基于 atom 组合 atom”

对应文件：

- `src/vanilla/utils/atomWithDefault.ts`
- `src/vanilla/utils/atomWithReset.ts`
- `src/vanilla/utils/selectAtom.ts`
- `src/vanilla/utils/splitAtom.ts`
- `src/vanilla/utils/unwrap.ts`
- `src/vanilla/utils/atomWithStorage.ts`
- `src/react/utils/useAtomCallback.ts`

结论：

> 很多高级能力不是加到 runtime 里，而是用 runtime 暴露的 atom 能力自己搭出来。

---

## 32. 架构图版：用 ASCII 图把三层关系画出来

### 32.1 总体分层图

```txt
+--------------------------------------------------+
|                    React App                     |
|--------------------------------------------------|
|  components                                      |
|    -> useAtom / useAtomValue / useSetAtom        |
+--------------------------|-----------------------+
                           |
                           v
+--------------------------------------------------+
|                 React binding layer              |
|--------------------------------------------------|
|  Provider / useStore / useAtomValue / useSetAtom |
|  作用：把 React 生命周期接到 store.get/set/sub     |
+--------------------------|-----------------------+
                           |
                           v
+--------------------------------------------------+
|                 Vanilla store runtime            |
|--------------------------------------------------|
|  store.get / store.set / store.sub               |
|  atomStateMap / mountedMap / changedAtoms        |
|  readAtomState / writeAtomState                  |
|  recomputeInvalidatedAtoms / flushCallbacks      |
+--------------------------|-----------------------+
                           |
                           v
+--------------------------------------------------+
|                    Atom configs                  |
|--------------------------------------------------|
|  primitive atom   derived atom   writable atom   |
|  init             read           write           |
+--------------------------------------------------+
```

### 32.2 atom 与 store 的关系图

```txt
atom config objects

  countAtom  ---------+
                      |
  priceAtom  ---------+----->  WeakMap<Atom, AtomState>  (per store)
                      |
  totalAtom  ---------+

AtomState example:

  totalAtom -> {
    d: Map([countAtom -> 3, priceAtom -> 7]),
    n: 10,
    v: 42,
    e: undefined,
    m: 18,
    p: Set(...)
  }
```

要点：

- atom 本身不带值
- store 才有每个 atom 的运行时状态
- 同一个 atom 可在不同 store 里对应不同 AtomState

### 32.3 依赖图示意

```txt
countAtom -----> doubledAtom -----> labelAtom
     \                               ^
      \-----> summaryAtom -----------|
```

这里可以理解成：

- `doubledAtom` 依赖 `countAtom`
- `summaryAtom` 也依赖 `countAtom`
- `labelAtom` 又依赖 `doubledAtom` 和 `summaryAtom`

当 `countAtom` 变化时：

1. `countAtom` 自己 epoch 变化
2. `doubledAtom / summaryAtom / labelAtom` 被标记 invalidated
3. 按依赖顺序重算

### 32.4 读流程图

```txt
Component render
   |
   v
useAtomValue(totalAtom)
   |
   v
store.get(totalAtom)
   |
   v
readAtomState(totalAtom)
   |
   +--> cache valid? -- yes --> return cached value
   |
   '--> no
         |
         v
      execute totalAtom.read(get)
         |
         +--> get(priceAtom)  -> readAtomState(priceAtom)
         |
         +--> get(countAtom)  -> readAtomState(countAtom)
         |
         '--> record dependencies in totalAtom.d
                and save new value / promise
```

### 32.5 写流程图

```txt
Component event
   |
   v
setCount(update)
   |
   v
store.set(countAtom, update)
   |
   v
writeAtomState(countAtom, update)
   |
   +--> update atomState.v
   +--> if changed: atomState.n++
   +--> changedAtoms.add(countAtom)
   '--> invalidateDependents(countAtom)
                |
                v
          recomputeInvalidatedAtoms()
                |
                v
            flushCallbacks()
                |
                v
         subscribed listeners run
                |
                v
          React components rerender
```

### 32.6 mounted 图示意

```txt
mountedMap

countAtom  -> { l: Set(listenerA), d: Set(), t: Set(doubledAtom) }
doubledAtom-> { l: Set(listenerB), d: Set(countAtom), t: Set(labelAtom) }
labelAtom  -> { l: Set(listenerC), d: Set(doubledAtom), t: Set() }
```

含义：

- `l` = 当前直接订阅它的组件监听器
- `d` = 当前挂载图中它依赖的 atom
- `t` = 当前挂载图中依赖它的 atom

这样 unmount 时就能知道：

- 这个 atom 只是暂时没人直接监听
- 还是已经彻底不再被任何活跃依赖链需要

### 32.7 async atom 流程图

```txt
useAtomValue(asyncAtom)
   |
   v
store.get(asyncAtom)
   |
   v
readAtomState(asyncAtom)
   |
   '--> atom.read(get, { signal }) returns Promise P1
                 |
                 v
           atomState.v = P1
                 |
                 v
         useAtomValue sees Promise
                 |
                 v
   createContinuablePromise(P1, () => store.get(asyncAtom))
                 |
                 v
          throw/use promise to Suspense

Later: dependency changes
   |
   v
asyncAtom recalculates -> Promise P2
   |
   v
continuablePromise switches to follow P2
   |
   v
P2 resolves -> rerender -> component gets final value
```

### 32.8 Provider / 多 store 关系图

```txt
                same atom config
                     |
         +-----------+-----------+
         |                       |
         v                       v
   Store A (Provider A)     Store B (Provider B)
         |                       |
         v                       v
 countAtom -> 1             countAtom -> 100
 userAtom  -> Alice         userAtom  -> Bob
```

要点：

- atom 是共享定义
- store 才是实例化后的状态空间
- 所以 Provider 的核心价值是“隔离 store”

---

## 33. 一份推荐的源码阅读顺序

如果你准备真正过源码，我建议按这个顺序：

1. `src/vanilla/atom.ts`
   - 先搞清楚 atom 到底是什么

2. `src/vanilla/store.ts`
   - 再理解公开 store API 很小

3. `src/vanilla/internals.ts`
   - 先看类型：`AtomState`、`Mounted`
   - 再看主线：
     - `ensureAtomState`
     - `readAtomState`
     - `writeAtomState`
     - `invalidateDependents`
     - `recomputeInvalidatedAtoms`
     - `storeSub / mountAtom / unmountAtom`
     - `flushCallbacks`

4. `src/react/Provider.ts`
   - 看 store 是怎么进入 React 的

5. `src/react/useSetAtom.ts`
   - 看写入桥接有多薄

6. `src/react/useAtomValue.ts`
   - 看订阅、rerender、Suspense

7. `src/vanilla/utils/*`
   - 重点看：
     - `atomWithDefault.ts`
     - `selectAtom.ts`
     - `splitAtom.ts`
     - `unwrap.ts`
     - `atomWithStorage.ts`

如果你按这个顺序走，会比较不容易在 `internals.ts` 里一上来就迷失。

---

## 34. 最后一句阅读建议

读这个 repo 时，尽量一直用下面这套映射去理解：

- `atom.ts`：定义层
- `store.ts`：API 入口层
- `internals.ts`：运行时执行层
- `react/*.ts`：视图绑定层
- `utils/*.ts`：组合扩展层

这样你会发现，Jotai 的核心其实非常统一：

> **用最小的 atom/store 抽象，搭出一个可缓存、可追踪依赖、可接 React、可扩展的细粒度状态图运行时。**

---

## 35. 关键函数逐段精读

这一节不再只讲“它做了什么”，而是讲“为什么要这么写”。

### 35.1 `BUILDING_BLOCK_readAtomState`

这是整个 runtime 的第一核心函数。

你可以把它分成 6 段来理解。

#### 第 1 段：拿到当前 atom 的 state 和上下文

它一开始会取出：

- `mountedMap`
- `invalidatedAtoms`
- `changedAtoms`
- `ensureAtomState`
- `mountDependencies`
- `setAtomStateValueOrPromise`
- `storeEpochHolder`

这说明 `readAtomState` 不只是“算值”，它还要同时处理：

- 缓存有效性
- 依赖关系维护
- 异步状态
- mount 图同步

#### 第 2 段：尽量走缓存

核心判断是：

1. 当前 atom 是否已初始化
2. 如果 mounted，且没 invalidated，可以直接复用
3. 如果没 mounted，但 store epoch 没变，也可复用
4. 否则再检查依赖的 epoch 是否有变化

这一段体现的是：

> **Jotai 的 read 默认是 memoized read，不是每次重新执行 read 函数。**

#### 第 3 段：构造内部 `getter`

这个 `getter(a)` 是理解 Jotai 的关键。

它做了几件事：

- 读取依赖 atom 的当前 state
- 返回依赖 atom 的值
- 把依赖 atom 记录进当前 atom 的 `d`
- 保存依赖当时的 epoch
- 如果当前 atom 已 mounted，还会维护 mounted 图上的反向依赖

也就是说，依赖追踪不是单独的一步，而是嵌在 `get(depAtom)` 调用里的。

#### 第 4 段：构造 `options.signal` 和 `options.setSelf`

这里是异步能力和内部自写能力的入口。

- `signal`：给 async read 取消旧请求用
- `setSelf`：允许 atom 在异步阶段给自己发写入

虽然 `setSelf` 现在已 deprecated，但从实现上能看出 Jotai 的思路：

- read 不是纯计算器那么简单
- 它也可以在某些场景和写入系统发生联动

#### 第 5 段：真正执行 `atomRead(store, atom, getter, options)`

这一步才会调用 atom 的 `read`。

然后根据结果：

- 如果是普通值，直接写进 atomState
- 如果是 Promise，就：
  - 把 Promise 也当成当前值保存
  - 注册 abort handler
  - 在 settle 后做依赖裁剪和 mount 同步

这一段很重要，因为它体现出：

> **Promise 在 Jotai 里首先是“值”，其次才是 Suspense 的输入。**

#### 第 6 段：finally 里处理“这次 read 是否导致 atom 变化”

`finally` 里会检查：

- 本次 read 后 `atomState.n` 是否变了
- 之前是否处于 invalidated 状态

如果是，就：

- 重新写回 invalidated 标记
- 放进 `changedAtoms`
- 触发 storeHooks.c

这意味着一个 read 过程本身也可能推动响应式图继续更新。

所以 `readAtomState` 的角色不是“纯 getter”，而是：

> **一个带缓存、依赖追踪、异步处理和图维护的求值过程。**

---

### 35.2 `BUILDING_BLOCK_writeAtomState`

这是第二核心函数。

也可以分成 4 段理解。

#### 第 1 段：构造 write 用的 `getter` / `setter`

- `getter(a)`：读取某个 atom 的当前值
- `setter(a, ...args)`：把写入递归派发给目标 atom

这个 `setter` 很关键，因为它统一了两种场景：

1. 写当前 atom 自己
2. 在一个 writable derived atom 里继续写别的 atom

也就是说，Jotai 把“action atom 调别人”也塞进了同一套写入框架里。

#### 第 2 段：如果写的是当前 primitive atom，直接落地

当 `a === atom` 时，会：

- 检查它是不是有初始值的可写 atom
- 更新当前值
- 调 `mountDependencies`
- 如有变化，推进 store epoch
- 加入 `changedAtoms`
- `invalidateDependents`

这里说明：

> **write 的真正落点仍然是更新 atomState，然后让依赖图失效。**

#### 第 3 段：如果写的是别的 atom，就递归 `writeAtomState`

这就是为什么 action atom 能层层 set 别的 atom。

你可以把它想成：

- atom write 不是“直接改字段”
- 而是“把命令继续发给另一个节点”

#### 第 4 段：异步写入后的统一重算与 flush

`finally` 里会根据 `isSync` 决定是否立即：

- `recomputeInvalidatedAtoms`
- `flushCallbacks`

这保证了：

- 同步链路里可以积累一批变化再一起提交
- 异步链路里则能在时机合适时继续刷新图

---

### 35.3 `BUILDING_BLOCK_recomputeInvalidatedAtoms`

这个函数解决的是一个非常具体的问题：

> 已经知道一批 atom 失效了，应该按什么顺序重算？

它分成两步。

#### 第一步：DFS 收集受影响节点，形成反向拓扑序

核心数据结构：

- `sortedReversedAtoms`
- `visiting`
- `visited`
- `stackAtoms`

它从 `changedAtoms` 出发，向所有 mounted 或 pending dependents 扩散。

重点不是“遍历所有 atom”，而是：

- 只遍历受影响子图
- 只处理活跃或异步相关依赖

#### 第二步：按逆序重算

之后再倒序处理这些 atom：

- 如果某个 atom 的依赖里确实有 changed atom
- 就重新 `readAtomState(store, a)`
- 并同步 `mountDependencies`

这里最妙的一点是：

- 不是无脑把 invalidated 的 atom 全部重算
- 而是只在“依赖真的变了”时重算

所以它既保持正确性，也保持了增量性。

---

### 35.4 `BUILDING_BLOCK_flushCallbacks`

这个函数像 runtime 的“提交阶段”。

它会循环处理三类东西：

- `changedAtoms` 对应 listener
- `mountCallbacks`
- `unmountCallbacks`

核心是这个 do-while：

- 收集本轮所有待执行 callback
- 执行它们
- 如果执行中又产生新的 changed/mount/unmount
- 再来一轮

也就是说：

> **Jotai 不是“变更后立刻通知一次”，而是“持续冲刷，直到图稳定”。**

这也是它能支撑链式副作用和异步重入的原因之一。

---

### 35.5 `useAtomValue`

React 绑定层里，最值得精读的是 `src/react/useAtomValue.ts`。

它可以分成 5 层。

#### 第 1 层：拿到 store

先通过 `useStore(options)` 确定当前组件到底连接哪个 store。

#### 第 2 层：用 `useReducer` 维护一个最小快照

保存的是三元组：

- `value`
- `store`
- `atom`

这样做比单纯 `useState(value)` 更稳，因为它还能检测：

- 这次 render 用的是不是同一个 store
- 用的是不是同一个 atom

#### 第 3 层：render 时同步读取一次 `store.get(atom)`

如果 reducer 里的快照与当前 store/atom 不一致，会立刻 rerender 并重新读取。

这能让 hook 在 store 切换或 atom 引用变化时保持一致。

#### 第 4 层：effect 里订阅 `store.sub(atom, listener)`

listener 被触发时：

- 如果值是 Promise，会先准备 promise status
- 然后调用 `rerender`

这里你可以看到一个很明确的边界：

- store 层负责触发 sub listener
- React hook 层负责把 listener 转成 rerender

#### 第 5 层：如果 value 是 Promise，就交给 Suspense

- `createContinuablePromise`
- `attachPromiseStatus`
- `use(promise)` / shim

这让 `useAtomValue` 同时扮演：

- 普通同步订阅 hook
- Suspense 数据读取 hook

---

## 36. 迷你实现版：一个帮助理解源码的简化 Jotai

下面这段代码不是对官方实现的精确复制，而是一个“足够像”的教学版。

目标只有 4 个：

1. 有 primitive atom
2. 有 derived atom
3. 有依赖追踪
4. 有订阅通知

```ts
import { useEffect, useState } from 'react'

type Atom<Value> = {
  init?: Value
  read: (get: <V>(atom: Atom<V>) => V) => Value
  write?: (
    get: <V>(atom: Atom<V>) => V,
    set: <V>(atom: Atom<V>, value: V) => void,
    update: unknown,
  ) => void
}

type AtomState<Value = unknown> = {
  value?: Value
  listeners: Set<() => void>
  dependents: Set<Atom>
}

const atomStateMap = new WeakMap<Atom, AtomState>()

function getAtomState<Value>(atom: Atom<Value>): AtomState<Value> {
  let state = atomStateMap.get(atom) as AtomState<Value> | undefined
  if (!state) {
    state = {
      value: atom.init,
      listeners: new Set(),
      dependents: new Set(),
    }
    atomStateMap.set(atom, state)
  }
  return state
}

export function atom<Value>(initialValue: Value): Atom<Value>
export function atom<Value>(
  read: (get: <V>(atom: Atom<V>) => V) => Value,
): Atom<Value>
export function atom<Value>(readOrInit: Value | ((get: any) => Value)) {
  if (typeof readOrInit === 'function') {
    return {
      read: readOrInit as (get: any) => Value,
    }
  }
  const config: Atom<Value> = {
    init: readOrInit,
    read: (get) => get(config),
    write: (_get, set, update) => {
      set(config, update as Value)
    },
  }
  return config
}

function readAtom<Value>(atom: Atom<Value>): Value {
  const state = getAtomState(atom)
  const get = <V,>(depAtom: Atom<V>) => {
    if (depAtom !== atom) {
      getAtomState(depAtom).dependents.add(atom)
      return readAtom(depAtom)
    }
    return state.value as Value
  }
  const value = atom.read(get)
  state.value = value
  return value
}

function notify(atom: Atom) {
  const state = getAtomState(atom)
  state.dependents.forEach((dep) => notify(dep))
  state.listeners.forEach((l) => l())
}

function writeAtom<Value>(atom: Atom<Value>, update: Value) {
  const state = getAtomState(atom)
  const get = <V,>(a: Atom<V>) => readAtom(a)
  const set = <V,>(a: Atom<V>, value: V) => {
    if (a === atom) {
      getAtomState(a).value = value
      notify(a)
      return
    }
    writeAtom(a, value)
  }
  atom.write?.(get, set, update)
  state.value = readAtom(atom)
}

export function useAtom<Value>(atom: Atom<Value>) {
  const [value, setValue] = useState(() => readAtom(atom))

  useEffect(() => {
    const state = getAtomState(atom)
    const callback = () => setValue(readAtom(atom))
    state.listeners.add(callback)
    callback()
    return () => state.listeners.delete(callback)
  }, [atom])

  return [value, (next: Value) => writeAtom(atom, next)] as const
}
```

这个迷你实现已经能让你看出 Jotai 主线：

- atom 是 config
- 用 WeakMap 存 atom state
- read 时收集依赖
- write 时通知 dependents
- hook 层只负责监听和 rerender

当然，和真实 Jotai 相比，它缺了很多关键能力：

- 没有 mounted/unmounted 图
- 没有缓存 epoch
- 没有 invalidation 与拓扑重算
- 没有 Promise / Suspense / abort
- 没有 store 隔离
- 没有 util 生态

但它已经足够解释“为什么真实实现会长成现在这样”。

---

## 37. 从迷你实现过渡到真实 Jotai，要补哪些东西

如果你已经能看懂上面的迷你版，那真实 Jotai 多出来的核心复杂度基本就是这 6 项：

### 37.1 从“dependents 集合”升级到“完整依赖表 + 版本号”

迷你版里只有：

- 谁依赖我

真实版里还有：

- 我依赖谁
- 依赖上次的 epoch 是多少
- 当前值的 epoch 是多少
- 当前 store 的 epoch 是多少

这一步解决的是：

- 精确缓存
- 避免重复重算
- 增量更新

### 37.2 从“递归 notify”升级到“失效传播 + 有序重算”

迷你版是：

- 直接递归通知 dependents

真实版是：

- 先 invalidation
- 再按依赖顺序 recompute
- 最后统一 flush listeners

这一步解决的是：

- 顺序正确
- 避免重复计算
- 支持链式变化

### 37.3 从“只关心 listeners”升级到“mounted 图”

迷你版只有 listener。

真实版还要维护：

- 当前挂载依赖
- 当前挂载 dependents
- onMount / onUnmount

这一步解决的是：

- 生命周期清理
- 按需保持依赖活跃
- 减少无用挂载

### 37.4 从“同步值”升级到“Promise 也是值”

迷你版默认所有值同步。

真实版把 Promise 当成一等公民：

- value 可以是 Promise
- Promise 被替换时可 abort
- React 层可接 Suspense

### 37.5 从“单例状态表”升级到“多 store 实例”

迷你版只有一个全局 `atomStateMap`。

真实版是：

- 每个 store 都有自己的 atomStateMap / mountedMap
- 同一个 atom 可以在多个 store 中并存不同状态

### 37.6 从“核心功能”升级到“组合式 util 生态”

迷你版只展示 runtime 核心。

真实版再往上叠加：

- storage
- reset
- default
- split
- select
- unwrap
- hydrate
- callback

这一步体现的是 Jotai 最重要的工程化价值：

> **核心很小，但表达力很强。**

---

## 38. 到这里为止，你应该形成的最终心智模型

如果你已经读到这里，我建议把 Jotai 最终压缩成下面这 8 句话：

1. `atom` 是配置对象，不是状态实体。
2. 真正的状态存在 store 里，而不是 atom 里。
3. store 用 `WeakMap<Atom, AtomState>` 管理运行时状态。
4. derived atom 的依赖关系是在 `read(get)` 时动态收集的。
5. 写入会推进版本号，并把下游依赖链标记为失效。
6. 重算是按依赖图顺序增量进行的，不是全量刷新。
7. React 层只负责把 `get/set/sub` 接到组件与 Suspense。
8. 大多数高级能力都是通过 atom 组合出来的，不是内核硬编码出来的。

如果把这 8 句话吃透，这个 repo 的主线你就已经真正掌握了。

---

## 39. 术语表：把源码里的缩写一次看懂

Jotai 的 `internals.ts` 里有不少短字段名，第一次看很容易晕。下面把它们翻成“人话”。

### 39.1 `AtomState` 字段表

`AtomState` 大致长这样：

- `d: Map<AnyAtom, EpochNumber>`
- `p: Set<AnyAtom>`
- `n: EpochNumber`
- `m?: EpochNumber`
- `v?: Value`
- `e?: Error`

逐个解释：

#### `d` = dependencies

含义：

- 当前 atom 依赖哪些 atom
- 以及依赖建立时，对方的 epoch 是多少

用途：

- 判断依赖有没有变化
- 决定缓存是否还能复用

可以理解为：

> “我的输入列表 + 每个输入当时的版本号”

#### `p` = pending promise dependents

含义：

- 当前 atom 有哪些依赖它的 atom 正在等待异步结果

用途：

- 异步场景下继续追踪受影响节点
- 让 Promise 链路替换后还能继续传播

可以理解为：

> “正在等我这个异步结果的下游节点集合”

#### `n` = atom epoch number

含义：

- 当前 atom 自己的版本号

什么时候变化：

- 值变了
- 错误变了
- Promise 被新 Promise 替换了

用途：

- 给依赖方做缓存比较
- 告诉系统“这个 atom 已经不是上次那个状态了”

#### `m` = last validated store epoch

含义：

- 当前 atomState 上次被确认有效时，store 的 epoch 是多少

用途：

- 在 atom 没有 mounted 时，也能做一层缓存判断

可以把它理解成：

> “这个 atomState 最近一次被认定是新鲜的，发生在 store 的哪个大版本上”

#### `v` = value

含义：

- 当前值

注意：

- 这个值可以是普通值
- 也可以是 Promise

#### `e` = error

含义：

- 当前 atom 的错误态

用途：

- `returnAtomValue` 时直接 throw
- 让 React error boundary 或调用方接住

---

### 39.2 `Mounted` 字段表

`Mounted` 大致长这样：

- `l: Set<() => void>`
- `d: Set<AnyAtom>`
- `t: Set<AnyAtom>`
- `u?: () => void`

逐个解释：

#### `l` = listeners

含义：

- 直接订阅这个 atom 的监听器集合

通常来自：

- `useAtomValue`
- `useAtom`
- `store.sub`

#### `d` = mounted dependencies

含义：

- 当前这个 mounted atom 在活跃依赖图里依赖哪些 atom

用途：

- mount 时递归挂载依赖
- 依赖变化时同步 mounted 图
- unmount 时知道要不要递归清理

#### `t` = mounted dependents / targets depending on me

含义：

- 当前活跃图里，哪些 atom 依赖我

用途：

- 失效传播
- unmount 时判断“我是否还被别人需要”

#### `u` = unmount cleanup

含义：

- 卸载时执行的清理函数

通常来自：

- atom 的 `onMount` 返回值

---

### 39.3 其他关键术语

#### `changedAtoms`

含义：

- 本轮变更中，真正已经发生值变化的 atom 集合

作用：

- 决定谁需要通知 listener
- 作为重算传播的起点

#### `invalidatedAtoms`

含义：

- 缓存已失效、需要重新校验或重算的 atom 集合

作用：

- 先标脏，再按需重算

#### `mountCallbacks`

含义：

- 待执行的 onMount 逻辑

#### `unmountCallbacks`

含义：

- 待执行的卸载 cleanup 逻辑

#### `storeEpochHolder`

含义：

- store 级别的大版本号容器

作用：

- 当某些写入发生时，整个 store 的“世界版本”推进
- 用于辅助判断缓存是否仍然可信

#### `abortHandlersMap`

含义：

- `Promise -> abort handlers` 的映射

作用：

- 当旧 Promise 过时，被新 Promise 替代时，可以统一触发 abort

#### `continuablePromise`

含义：

- React 层为了 Suspense 构造的“可续接 Promise”

作用：

- 当前等待的 Promise 过时后，切到新的 Promise 继续等
- 避免组件卡死在旧 Promise 上

---

### 39.4 一眼辨认表

如果你只想快速记忆，可以记这个表：

```txt
AtomState
  d = 我依赖谁
  p = 谁在异步等我
  n = 我的版本号
  m = 我上次验证时 store 的版本
  v = 我的值
  e = 我的错误

Mounted
  l = 谁直接监听我
  d = 我在活跃图里依赖谁
  t = 活跃图里谁依赖我
  u = 我卸载时要做什么清理
```

---

## 40. 常见面试题版：把实现原理转成回答模板

下面这些问题，几乎就是“这个 repo 实现原理”的口语化版本。

### 40.1 Jotai 为什么不需要字符串 key？

短答：

- 因为 Jotai 直接用 atom 对象本身做 identity
- store 里通过 `WeakMap<Atom, AtomState>` 保存状态
- 所以不需要像 Recoil 一样维护全局字符串 key

扩展答法：

- 这样避免了命名冲突
- 同一个 atom 定义可在多个 store 中各自持有状态
- 还更利于垃圾回收，因为 WeakMap key 没引用后可回收

---

### 40.2 为什么说 atom 不存值，store 才存值？

短答：

- atom 是配置对象，只描述 `init / read / write / onMount`
- 真正的运行时值在 store 的 `atomStateMap` 里

扩展答法：

- 这让同一个 atom 配置可以在多个 store 中实例化出多份状态
- 也让 React 层和状态引擎层解耦

---

### 40.3 derived atom 的依赖关系是怎么建立的？

短答：

- 在执行 `atom.read(get)` 时动态建立
- 每次 `get(depAtom)`，Jotai 都会记录：
  - 依赖的是谁
  - 依赖当时的 epoch 是多少

扩展答法：

- 所以依赖图不是静态声明的，而是运行时求值得到的
- 这也让依赖关系可以随条件分支动态变化

---

### 40.4 Jotai 为什么能做到细粒度更新？

短答：

- 因为它按 atom 粒度建图和订阅
- 写入后只让受影响的依赖链 invalidated 并重算
- 不相关 atom 和组件不会被牵连

扩展答法：

- 这和“大 store + selector”模型不一样
- 它不是先整个 store 变，再让 selector 去筛选
- 而是天然以 atom 为最小更新单元

---

### 40.5 Jotai 为什么需要 `Provider`，它不是可选的吗？

短答：

- 对，`Provider` 不是语法上必须的，因为有默认 store
- 但它在工程上很重要，因为它决定使用哪个 store

扩展答法：

- Provider 带来状态隔离
- 同一套 atom 可以在不同 Provider 子树里有不同值
- SSR 场景下更关键，因为默认 store 可能跨请求共享

---

### 40.6 Jotai 的 async atom 为什么能接 Suspense？

短答：

- 因为 async atom 本质是“值为 Promise 的 atom”
- store 层把 Promise 当成普通值保存
- React 层在 `useAtomValue` 里识别 Promise，并交给 `use()` / Suspense

扩展答法：

- 同时还有 `continuablePromise` 来处理 Promise 替换
- 以及 abort handler 来取消过时异步请求

---

### 40.7 Jotai 为什么既要 `atomStateMap` 又要 `mountedMap`？

短答：

- `atomStateMap` 管值和依赖缓存
- `mountedMap` 管活跃订阅图和生命周期

扩展答法：

- 一个 atom 即使没组件订阅，也可能被 `store.get` 或别的 atom 读取
- 所以它需要 state，但不一定需要 mounted 信息
- 拆开后能更好做缓存与清理

---

### 40.8 `invalidateDependents` 和 `recomputeInvalidatedAtoms` 分别负责什么？

短答：

- `invalidateDependents`：先把下游节点标脏
- `recomputeInvalidatedAtoms`：再按依赖顺序重算真正受影响节点

扩展答法：

- 一个负责“传播失效”
- 一个负责“有序求值”
- 这比简单递归重算更稳定，也更高效

---

### 40.9 Jotai 的 util 为什么很多都能在用户层实现？

短答：

- 因为 Jotai 的 atom 抽象表达力很强
- 一个 atom 可以是值节点、计算节点、动作节点
- 所以很多高级功能都能通过 atom 组合出来

扩展答法：

- 例如 `selectAtom` 是“selector + equality”的组合
- `atomWithDefault` 是“默认派生 + 覆盖值”的组合
- `atomWithStorage` 是“普通 atom + 外部持久化同步”的组合

---

### 40.10 Jotai 和 Redux / Zustand 的实现风格差别是什么？

短答：

- Redux / Zustand 更偏“大 store 里取 slice”
- Jotai 更偏“以 atom 为节点构建细粒度依赖图”

扩展答法：

- Redux 擅长明确的数据流与 reducer 组织
- Zustand 擅长中心 store + selector 订阅
- Jotai 则更像响应式图求值系统

---

### 40.11 如果让你自己实现一个最小版 Jotai，你会先做什么？

短答：

1. `atom(init)`
2. `WeakMap<atom, state>`
3. `get / set / subscribe`
4. derived atom 的 `read(get)`
5. 依赖追踪
6. 写入通知下游

扩展答法：

- 再往后补 epoch 缓存
- 再补 invalidation + 有序重算
- 再补 mounted 图
- 最后补 Promise/Suspense 和各种 util

---

## 41. 再进一步：如果你要把它讲给别人，最稳的讲法

如果面试、分享或 code review 里要用最稳的方式描述 Jotai，我建议按这个顺序说：

### 第一句：先定性

> Jotai 是一个以 atom 为最小节点的细粒度状态图，而不是一个大对象 store 的 selector 系统。

### 第二句：再说运行时

> atom 自己不存值，真正的值和依赖缓存都存放在 store 的 `WeakMap<Atom, AtomState>` 里。

### 第三句：说依赖建立

> derived atom 在执行 `read(get)` 时动态收集依赖，并记录依赖版本号。

### 第四句：说更新机制

> 写入发生后，Jotai 不会全量刷新，而是先标记下游失效，再按依赖图顺序增量重算。

### 第五句：说 React 层

> React 层很薄，只是把 `store.get/set/sub` 接到 hook、rerender 和 Suspense 上。

### 第六句：说扩展能力

> 很多高级功能都不是内核特殊分支，而是通过 atom 组合出来的 util。

如果能把这 6 句讲顺，你基本就已经能把这个 repo 讲清楚了。

---

## 42. 最终压缩版

如果最后还要再压缩一次，我会把 Jotai 归纳成这个公式：

```txt
Jotai =
  atom config
+ per-store runtime state
+ dynamic dependency tracking
+ incremental recomputation
+ thin React binding
+ composable utilities
```

翻成中文就是：

> **Jotai = atom 配置对象 + 每个 store 的运行时状态 + 动态依赖追踪 + 增量重算 + 很薄的 React 绑定层 + 可组合的工具生态。**

---

## 43. 源码导读目录版：按文件看时，应该盯什么

如果你要真正“走读”这个 repo，最容易的方法不是从头到尾看，而是给每个文件设置一个明确的问题。

---

### 43.1 `src/index.ts`

你要问自己的问题：

- 这个包最顶层对外暴露了什么？

看点：

- 它只是 re-export：`./vanilla.ts` + `./react.ts`

你应该得到的结论：

> 顶层 API 只是把核心层和 React 层拼起来，没有额外逻辑。

---

### 43.2 `src/vanilla.ts`

你要问自己的问题：

- 不依赖 React 的最小 public API 是什么？

看点：

- `atom`
- `createStore`
- `getDefaultStore`
- 类型导出

你应该得到的结论：

> Jotai 的核心其实是一个跟 React 解耦的 atom/store runtime。

---

### 43.3 `src/react.ts`

你要问自己的问题：

- React 层到底比 vanilla 多了什么？

看点：

- `Provider`
- `useAtomValue`
- `useSetAtom`
- `useAtom`

你应该得到的结论：

> React 层做的事情非常少，本质就是把 store 接到 hook。

---

### 43.4 `src/vanilla/atom.ts`

你要问自己的问题：

- atom 到底是什么数据结构？
- primitive / derived / writable 是怎么统一起来的？

重点看：

- `export function atom(...)` 的几个 overload
- `defaultRead`
- `defaultWrite`

阅读顺序建议：

1. 先看类型定义：`Atom`、`WritableAtom`、`PrimitiveAtom`
2. 再看 `atom(...)` 的实现
3. 最后看 `defaultRead/defaultWrite`

你应该得到的结论：

> atom 只是一个配置对象，Jotai 通过不同字段把几种 atom 统一到一个抽象里。

---

### 43.5 `src/vanilla/store.ts`

你要问自己的问题：

- store public API 长什么样？
- 默认 store 是怎么来的？

重点看：

- `createStore`
- `getDefaultStore`
- `INTERNAL_overrideCreateStore`

阅读顺序建议：

1. 先看 `createStore`
2. 再看 `getDefaultStore`
3. 再理解默认 store 的 dev warning

你应该得到的结论：

> store 对外接口很小，但 provider-less 模式就是靠默认 store 实现的。

---

### 43.6 `src/vanilla/internals.ts`

这是重头戏。你不要一口气通读，建议拆开看。

#### 第一遍：先只看类型

重点看：

- `type AtomState`
- `type Mounted`
- `type BuildingBlocks`

你的目标不是理解所有细节，而是先知道：

- store 里到底有哪些状态桶
- runtime 被拆成了哪些 building block

#### 第二遍：再看最小主线

只看这些函数：

- `BUILDING_BLOCK_ensureAtomState`
- `BUILDING_BLOCK_readAtomState`
- `BUILDING_BLOCK_writeAtomState`
- `BUILDING_BLOCK_storeGet`
- `BUILDING_BLOCK_storeSet`
- `BUILDING_BLOCK_storeSub`

你的问题应该是：

- 读怎么发生
- 写怎么发生
- 订阅怎么发生

#### 第三遍：再看图维护

重点看：

- `BUILDING_BLOCK_invalidateDependents`
- `BUILDING_BLOCK_recomputeInvalidatedAtoms`
- `BUILDING_BLOCK_mountDependencies`
- `BUILDING_BLOCK_mountAtom`
- `BUILDING_BLOCK_unmountAtom`
- `BUILDING_BLOCK_flushCallbacks`

你的问题应该是：

- 失效怎么传播
- 为什么重算是有序的
- 生命周期怎么管理

#### 第四遍：最后看异步增强

重点看：

- `BUILDING_BLOCK_setAtomStateValueOrPromise`
- `BUILDING_BLOCK_registerAbortHandler`
- `BUILDING_BLOCK_abortPromise`

你的问题应该是：

- Promise 为什么可以直接成为 atom value
- 旧 Promise 为什么能被中断

你应该得到的结论：

> `internals.ts` 本质上是在实现一个可增量求值、可订阅、可异步中断的状态图执行器。

---

### 43.7 `src/react/Provider.ts`

你要问自己的问题：

- store 是怎么进入 React tree 的？

重点看：

- `StoreContext`
- `useStore`
- `Provider`

你应该得到的结论：

> Provider 的关键不是“包一层 Context”，而是“决定当前组件树绑定哪个 store”。

---

### 43.8 `src/react/useSetAtom.ts`

你要问自己的问题：

- set hook 到底有多少逻辑？

重点看：

- `useCallback(... => store.set(atom, ...args))`

你应该得到的结论：

> 写入桥接层非常薄，复杂度不在这里。

---

### 43.9 `src/react/useAtom.ts`

你要问自己的问题：

- `useAtom` 自己做了什么？

重点看：

- 它几乎只是 `useAtomValue + useSetAtom`

你应该得到的结论：

> 真正复杂的是 value read，不是 tuple 组合。

---

### 43.10 `src/react/useAtomValue.ts`

你要问自己的问题：

- React rerender 是怎么接上 store 订阅的？
- Promise 是怎么接到 Suspense 的？

重点看：

- `useReducer` 快照
- `useEffect(() => store.sub(...))`
- `isPromiseLike`
- `createContinuablePromise`
- `use(promise)` shim

阅读顺序建议：

1. 先看 `useReducer` 保存什么
2. 再看 effect 订阅
3. 最后看 Promise 路径

你应该得到的结论：

> `useAtomValue` 是 React 层真正的核心，它把“外部 store 订阅”和“React Suspense 读取”缝在一起。

---

### 43.11 `src/react/utils/useHydrateAtoms.ts`

你要问自己的问题：

- hydration 是不是一种特殊的 store 初始化协议？

重点看：

- `hydratedMap`
- 对 values 的遍历
- `store.set(atom, ...args)`

你应该得到的结论：

> hydration 实际上就是“带去重保护的批量 set”。

---

### 43.12 `src/react/utils/useAtomCallback.ts`

你要问自己的问题：

- imperative 访问 atom 是怎么做出来的？

重点看：

- `useMemo(() => atom(null, writeFn), [callback])`
- `useSetAtom(anAtom)`

你应该得到的结论：

> 即使是 imperative callback，Jotai 也尽量复用 atom 自己的 write 模型来实现。

---

### 43.13 `src/vanilla/utils/selectAtom.ts`

你要问自己的问题：

- selector 风格能力是怎么通过 atom 表达出来的？

重点看：

- `memo3`
- `prevSlice`
- `equalityFn`
- `derivedAtom.init = EMPTY` 这个 hack

你应该得到的结论：

> `selectAtom` 本质是“派生 atom + slice memo + 稳定缓存”。

---

### 43.14 `src/vanilla/utils/splitAtom.ts`

你要问自己的问题：

- 为什么一个数组 atom 能拆成一组 item atom？

重点看：

- `mappingCache`
- `getMapping`
- item atom 的 `read/write`
- dispatch action: `remove/insert/move`

你应该得到的结论：

> `splitAtom` 的本质是“把数组索引/键映射包装成一组子 atom”。

---

### 43.15 `src/vanilla/utils/unwrap.ts`

你要问自己的问题：

- async atom 如何被变成同步可读值？

重点看：

- `promiseErrorCache`
- `promiseResultCache`
- `refreshAtom`
- `triggerRefreshAtom.INTERNAL_onInit`

你应该得到的结论：

> `unwrap` 是通过额外 atom 驱动刷新，把 Promise 生命周期转成同步状态视图。

---

### 43.16 `src/vanilla/utils/atomWithStorage.ts`

你要问自己的问题：

- 外部存储同步是怎么嫁接进 atom 的？

重点看：

- `createJSONStorage`
- `baseAtom`
- `baseAtom.onMount`
- write 中的 `storage.setItem/removeItem`

你应该得到的结论：

> 持久化不是核心能力，而是“atom + onMount + storage adapter”的组合。

---

## 44. 易错点 / 坑点版：读源码和使用时最容易踩的地方

这一节一半是源码阅读坑，一半是实际使用坑。

---

### 44.1 在 render 里新建 atom，可能导致无限循环或重复订阅

典型问题：

```ts
function Comp() {
  const [value] = useAtom(atom(0))
  return <div>{value}</div>
}
```

为什么有问题：

- 每次 render 都会创建一个新的 atom 对象
- 对 Jotai 来说，这就是一个全新的 identity
- 于是 store 会把它当成一个全新的状态节点

结果：

- 缓存失效
- 订阅反复重建
- 某些情况下会陷入无限循环

正确方式：

- 把 atom 提到组件外
- 或用 `useMemo` 保持引用稳定

---

### 44.2 `selectAtom` 必须保证输入引用稳定

典型问题：

```ts
const [value] = useAtom(selectAtom(baseAtom, (v) => v.foo))
```

如果这行直接写在 render 里，风险在于：

- `selectAtom(...)` 每次 render 都可能创建新 atom
- selector 函数若也是新引用，memo 缓存也打不中

后果：

- 不必要重建
- 订阅抖动
- 极端情况下导致循环

正确方式：

- `baseAtom` 稳定
- selector 函数稳定
- 整个 `selectAtom(...)` 结果最好也稳定，比如 `useMemo`

---

### 44.3 默认 store 在 SSR 有风险

问题本质：

- `getDefaultStore()` 是进程级共享的默认 store
- 服务端多请求可能共用它

后果：

- 用户 A 请求写进去的状态，理论上可能污染用户 B
- 会带来逻辑错误和安全风险

正确方式：

- SSR 时用 `Provider`
- 每个请求创建自己的 store

---

### 44.4 多个 Jotai 实例并存会报警

你在 `getDefaultStore()` 里能看到 dev warning：

- `Detected multiple Jotai instances...`

为什么：

- 如果 bundle 里有多个 Jotai 副本
- 每个副本都会有自己的默认 store
- 但你的 React tree / atoms / hooks 可能跨实例交错

后果：

- 默认 store 语义混乱
- 调试困难
- 行为不符合预期

常见诱因：

- monorepo 依赖重复安装
- link / workspace 配置不当
- 某个包锁定了不同版本

---

### 44.5 async atom 和某些 util 组合时要注意同步/异步边界

问题本质：

- 有些 util 期望拿到同步 atom
- 但 async atom 的值可能是 Promise

典型场景：

- `splitAtom`
- 一些第三方扩展 util

正确方式：

- 先 `unwrap` 或用别的方式把 async atom 转成同步视图
- 再交给这些 util

核心理解：

> async atom 在 runtime 里是“正常 atom”，但不是所有 util 都愿意直接处理 Promise 值。

---

### 44.6 `atomWithStorage` 的初始化时机要搞清楚

容易误解的点：

- 有 `getOnInit` 时，初始化值会直接从 storage 读
- 没有时，先用 `initialValue`，挂载后再同步 storage 值

后果：

- 如果你没搞清楚，会觉得“为什么首屏先是初始值，之后又跳一下”

本质上这是一个 hydration / 挂载时机问题，不是 bug。

---

### 44.7 `atomWithDefault` 一旦被覆盖，就会临时脱离默认派生关系

很多人第一次用会误会：

- 默认值是从别的 atom 派生出来的
- 那我改了它以后，原依赖变化是不是还会继续推着它变？

答案是：

- 不一定
- 因为一旦写入覆盖值，它会优先读 `overwrittenAtom`
- 这时默认计算分支就暂时不再参与了

如果你想回到默认派生关系，需要 reset 到初始逻辑。

---

### 44.8 action atom 里可以 set 很多 atom，但要小心可读性

Jotai 允许你这样写：

```ts
const saveAtom = atom(null, (get, set) => {
  set(aAtom, ...)
  set(bAtom, ...)
  set(cAtom, ...)
})
```

这很灵活，但潜在问题是：

- 依赖关系会变得分散
- 写入链路难以追踪
- 调试时不如 reducer 集中清晰

所以这是“强能力”，但不是“随便滥用更好”。

---

### 44.9 `onMount` 很强，但它是生命周期钩子，不是业务逻辑垃圾桶

容易踩的坑：

- 把很多复杂业务副作用都堆进 `onMount`

风险：

- 状态初始化变得隐式
- mount/unmount 行为难推理
- 在 Provider 切换、条件渲染、SSR 场景下更难分析

更稳的做法：

- 用它做“订阅外部源 / 注册监听 / 必要初始化”
- 复杂业务流程仍尽量放在显式 action atom 或组件逻辑里

---

### 44.10 `useHydrateAtoms` 默认只 hydrate 一次

很多人会误会：

- 我 rerender 时传了新初始值，为什么 atom 没更新？

原因：

- `useHydrateAtoms` 内部会记住这个 store 已经 hydrate 过哪些 atom
- 默认情况下同一个 store 只 hydrate 一次

如果你真的要强制重灌：

- 用 `dangerouslyForceHydrate`

但通常更推荐：

- 明确地 `store.set`
- 或在 store 生命周期层面重新创建 store

---

### 44.11 不要把 Jotai 想成“随便读写都便宜”的黑盒

虽然 Jotai 很轻，但不意味着：

- 动态造很多 atom 没代价
- 在 render 里频繁组装新 atom 没代价
- 复杂依赖图不会影响调试成本

更准确的理解是：

- Jotai 给了你很高的表达力
- 但你仍然需要管理引用稳定性、依赖复杂度和状态边界

---

### 44.12 读源码时不要被 `BuildingBlocks` 吓到

`internals.ts` 里把很多东西都塞进了 `BuildingBlocks` tuple，看起来很绕。

其实阅读时不要先关心“为什么是 tuple 而不是对象”，而应该先关心：

- 这里到底把 runtime 拆成了哪些职责块
- 哪些是状态桶
- 哪些是主算法
- 哪些只是胶水函数

一旦把职责看明白，这个 tuple 写法就只是实现细节，不再是障碍。

---

## 45. 这份文档最后给你的建议

如果你接下来真的要把这个 repo 吃透，我建议这么做：

1. 先把第 2、4、5、6、7、8、10、11 节再读一遍
2. 再看第 31 节源码标注版
3. 然后按第 43 节源码导读目录，自己过一遍文件
4. 遇到短字段名再回来看第 39 节术语表
5. 想验证自己是否理解了，就用第 40、41 节的问答模板自己复述
6. 最后再看第 36、37 节的迷你实现版，把真实实现映射回简化模型

如果你能走完这条路径，基本就不只是“知道 Jotai 怎么用”，而是真的理解了它为什么这样实现。
