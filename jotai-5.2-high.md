# Jotai 5.2（当前仓库）实现原理

> 本文基于当前仓库 `src/` 下的实现做源码导向的说明：vanilla 层负责 store/依赖图/调度，react 层负责订阅与 Suspense 适配。

## 1. 模块分层与入口

- 对外入口：`src/index.ts`（导出 `src/vanilla.ts` 与 `src/react.ts`）。
- Vanilla 核心：
  - `src/vanilla/atom.ts`：atom 工厂与基础 `read`/`write` 约定。
  - `src/vanilla/store.ts`：`createStore`/`getDefaultStore`，内部调用 `src/vanilla/internals.ts` 构建 store。
  - `src/vanilla/internals.ts`：依赖跟踪、缓存、写入传播、挂载/卸载、异步 Promise 管理（核心实现）。
- React 集成：
  - `src/react/Provider.ts`：通过 Context 注入 store。
  - `src/react/useAtomValue.ts`：订阅 store + 读取值（含 Promise/Suspense 处理）。
  - `src/react/useSetAtom.ts`：封装 `store.set`。
  - `src/react/useAtom.ts`：组合读写 hooks。

## 2. Atom：统一读写协议与“原子化”

- Atom 在本仓库里不是“类”，而是一个包含 `read`（必需）和 `write`（可选）的配置对象。
- `atom(...)` 工厂（`src/vanilla/atom.ts`）支持三类常见形态：
  - 基础 atom：保存一个值（带 `init`），默认 `read` 返回自身，默认 `write` 支持 setState 风格（值或 updater）。
  - 只读派生 atom：仅提供 `read(get, options)`。
  - 可写派生 atom：提供 `read` + `write(get, set, ...args)`，把“业务写入”编排为对其它 atom 的 `set`。

## 3. Store 的内部状态：AtomState 与 Mounted

store 以 `WeakMap` 为主存储（便于 GC），核心有两类状态（`src/vanilla/internals.ts`）：

- AtomState（所有 atom 都可能有，挂载与否无关）：
  - `d: Map<Atom, Epoch>`：该 atom 的依赖集合，Map 的 value 是依赖在计算时的 epoch（版本号）。
  - `p: Set<Atom>`：依赖该 atom 且当前处于 pending promise 的 atom（用于异步传播）。
  - `n: Epoch`：该 atom 自身的 epoch（值变化就递增）。
  - `v`/`e`：当前值或错误。
- Mounted（仅在 atom 被订阅或作为订阅 atom 的传递依赖时存在）：
  - `l: Set<() => void>`：订阅监听（组件 rerender 回调）。
  - `d: Set<Atom>`：挂载态的依赖集合。
  - `t: Set<Atom>`：依赖当前 atom 的“上游”集合（反向边）。
  - `u?: () => void`：onMount 的卸载函数。

## 4. 读取：惰性计算 + 依赖追踪 + 缓存复用

读取最终都会进入 `readAtomState(store, atom)`：

- 先尝试复用缓存：
  - 若 atom 已初始化且已挂载，并且未被标记 invalidated，则直接返回缓存。
  - 否则检查其依赖 `d`：依赖的 epoch 若都没变，也可复用缓存。
- 需要重算时：
  - 清空旧依赖 `d`，执行 `atom.read(getter, options)`。
  - `getter(depAtom)` 在返回依赖值的同时，会把 `depAtom` 写入 `d` 并记录 `depAtom` 的 epoch；若当前 atom 是 mounted，则同时维护 mounted 反向依赖（`depMounted.t.add(atom)`）。
  - 计算产物（值/Promise）通过 `setAtomStateValueOrPromise` 写回 AtomState；如果是 Promise，会注册 abort handler 并维护 pending 关系。

结果：依赖图是“在 read 过程中动态收集”的，每次 read 都能得到精确依赖集合（可随分支变化）。

## 5. 写入：changed 集合 + 失效传播 + 批量 flush

写入入口为 `store.set(...)`，内部调用 `writeAtomState(store, atom, ...args)`：

- 对“基础 atom 写自身”的特殊路径：直接更新值并递增 epoch，把该 atom 加入 `changedAtoms`，并 `invalidateDependents` 让所有上游派生 atom 进入 invalidated。
- 对“派生 atom 的 write”：执行 `atom.write(get, set, ...args)`，其内部通过 `set(...)` 触发对其它 atom 的更新；结束后统一进行重新计算与通知。

写入完成后，`store.set` 会强制：

1) `recomputeInvalidatedAtoms(store)`：基于 changed 集合拓扑化遍历依赖图，只重算受影响的 atom。
2) `flushCallbacks(store)`：把变更通知（订阅回调、mount/unmount 回调等）批处理执行。

## 6. 挂载/卸载：把“依赖图”限制在使用中的子图

- `store.sub(atom, listener)` 会 `mountAtom`：
  - 先读一次 atom 得到其依赖，再递归挂载依赖（建立 mounted 反向边）。
  - 将 listener 放入 `Mounted.l`。
  - 触发 flush，使 mount/onMount 回调在批处理阶段执行。
- 取消订阅后：若 atom 没有 listener 且不再作为任何 mounted atom 的依赖，则 `unmountAtom` 会递归卸载其依赖，并调用 onUnmount。

## 7. 异步与 Promise：可中止与“continuable”

- Vanilla 层用 `promiseStateMap` 跟踪 promise 是否 pending，以及关联的 abort handlers；当一个 atom 的值从旧 Promise 切到新值时，会尝试 abort 旧 Promise。
- React 层（`src/react/useAtomValue.ts`）为了适配 Suspense/旧 React：
  - 对 Promise 值构造“continuable promise”，当旧 promise 被 abort 时，会尝试用 `store.get(atom)` 获取新值/新 promise 并续上；
  - React 19+ 有 `React.use(promise)` 时直接用，否则用 shim（通过给 promise 附加 `status/value/reason`）。

## 8. React 订阅模型：最小化 rerender

- `useAtomValue` 通过 `useReducer` 存储 `[value, store, atom]`，保证：
  - store/atom 未变化且 `Object.is(prevValue, nextValue)` 时不触发 state 更新；
  - 订阅回调触发时调用 `rerender()`，从而让组件仅在 atom 值变化时更新。
- `useSetAtom` 提供稳定的 `store.set` 回调。
- `useAtom` 只是把两者组合成 `[value, set]`。

