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

## 17. 总结

**Jotai 的实现原理就是：**

- 用 `atom` 对象描述状态单元和依赖关系
- 用 `WeakMap` 在 `store` 中保存每个 atom 的运行时状态
- 通过依赖收集、版本号和失效传播实现精细更新
- 用很薄的 React 层把 store 和组件连接起来
- 用 atom 组合 atom 的方式扩展出大量高级能力

一句话总结：

> **Jotai 不是“一个大 store + selector 系统”，而是“一个以 atom 为节点的细粒度响应式状态图”。**
