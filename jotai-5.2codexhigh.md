# 当前仓库（Jotai）实现原理说明（高层 + 关键细节）

> 说明：本文基于当前仓库源码（`src/vanilla` 与 `src/react`）进行梳理，侧重数据结构、依赖追踪、更新调度与 React 集成。

## 1. 核心分层与目录结构

- **vanilla 核心层**：`src/vanilla/*`
  - `atom.ts`：定义 atom 工厂、读写协议与类型。
  - `internals.ts`：Store 内核、依赖图与调度系统。
  - `store.ts`：对外的 store 入口（`createStore` / `getDefaultStore`）。
- **React 集成层**：`src/react/*`
  - `Provider`、`useAtomValue`、`useSetAtom`、`useAtom` 等 hooks。
- **工具与扩展**：`src/vanilla/utils/*`、`src/react/utils/*`
  - 基于核心 atom 能力封装派生行为，如 `atomFamily`、`atomWithStorage`、`selectAtom` 等。
- **构建与调试**：`src/babel/*`
  - debugLabel 插件等，辅助开发期可读性与热更新体验。

## 2. Atom 设计：最小状态与统一读写协议

**Atom 是一个可被 Store 解释的“读写协议对象”**，核心字段为：

- `read(get, options)`：读取值，并通过 `get(otherAtom)` 建立依赖图。
- `write(get, set, ...args)`：可选写逻辑（派生 atom 的更新入口）。
- `onMount(setAtom)`：被首次挂载时触发，返回 `onUnmount`。

`atom()` 工厂会生成：

- **primitive atom**：带初始值，默认 `read` 直接读自身，`write` 默认 `set` 自身。
- **derived atom**：由 `read` 计算依赖；若提供 `write` 则可更新。

每个 atom 都有唯一 `key`，`toString()` 在开发态支持 `debugLabel` 以便调试。

## 3. Store 的内部核心：AtomState 与 Mounted

Store 内部通过 WeakMap 保存状态以便 GC：

**AtomState（无论是否挂载都会存在）**

- `d: Map<atom, epoch>`：依赖图与依赖的版本号。
- `p: Set<atom>`：依赖该 atom 且处于 pending Promise 的 atom。
- `n: epoch`：当前版本号。
- `v` / `e`：当前值 / 错误。

**Mounted（仅已挂载 atom）**

- `l: Set<listener>`：订阅监听。
- `d: Set<atom>`：挂载态依赖集合。
- `t: Set<atom>`：依赖它的上游集合。
- `u: onUnmount`：卸载回调。

此外还维护：

- `invalidatedAtoms`：失效原子与其版本号。
- `changedAtoms`：变更原子集合。
- `mountCallbacks/unmountCallbacks`：延迟执行的生命周期回调。
- `storeHooks`：实验性 hooks（i/r/c/m/u/f）用于调试或扩展。

## 4. 读取流程：缓存、依赖追踪与 Promise

读取入口为 `readAtomState`，流程要点：

1. **缓存判断**
   - 若 atom 已初始化且未失效，直接返回。
   - 否则检查依赖的 epoch 是否变化，若未变化则复用缓存。

2. **重新计算**
   - 清空旧依赖 `d`。
   - 执行 `atom.read(get, options)`。
   - `get` 会记录依赖与其 epoch，并在需要时挂载依赖关系。

3. **Promise 处理**
   - 若值为 Promise：
     - 注册 abort handler（新值覆盖旧 Promise 时可中止旧 Promise）。
     - 将 pending 关系写入依赖原子的 `p` 集合。

4. **setSelf（deprecated）**
   - `options.setSelf` 仅允许异步读取后的写入，且开发态会发出警告。

## 5. 写入流程：最小影响范围更新

写入入口为 `writeAtomState` / `store.set`：

- **写 primitive atom**：
  - 直接更新自身 `v`，递增 `n`。
  - 标记为 `changed`，并 `invalidateDependents` 使依赖链失效。

- **写 derived atom**：
  - 执行自定义 `atom.write(get, set, ...args)`。
  - 支持在一次写入中更新多个 atom。

写入后由 `recomputeInvalidatedAtoms` 与 `flushCallbacks` 统一处理重新计算与通知。

## 6. 失效传播与拓扑重算

**失效传播（invalidateDependents）**

- 沿“挂载依赖 + pending 依赖”向上递归。
- 被影响原子写入 `invalidatedAtoms`。

**重新计算（recomputeInvalidatedAtoms）**

- 从 `changedAtoms` 出发，构建依赖图的拓扑序（DFS）。
- 只对依赖发生变化的 atom 重新计算。
- 重算后更新挂载依赖集合，清理失效状态。

核心目标是：**最小化重算范围**。

## 7. 挂载/卸载与订阅机制

- `store.sub(atom, listener)` 会触发 `mountAtom`：
  - 递归挂载依赖。
  - 注册 `onMount` 回调并延迟执行。
- `unmountAtom` 会在 **无订阅且不再被依赖** 时触发：
  - 执行 `onUnmount`。
  - 递归卸载依赖。

挂载策略确保依赖图只包含实际使用中的 atom 子图。

## 8. 异步与 Abortable Promise 模型

- `promiseStateMap` 记录 Promise 的 pending 状态与 abort handler。
- 当新值覆盖旧 Promise：
  - 旧 Promise 的 abort handler 会执行。
- pending 依赖关系通过 `AtomState.p` 维护，保证异步链上的依赖能正确失效与重算。

## 9. React 集成：Provider 与 Hooks

**Provider / useStore**

- `Provider` 可注入自定义 store，否则内部 `createStore()`。
- `useStore` 从 Context 取 store 或使用默认 store。

**useAtomValue**

- `useReducer` 缓存 `[value, store, atom]`，避免无效 rerender。
- 订阅 `store.sub`；在变更时触发 rerender。
- 对 Promise 值构造 *continuable promise*，支持 Suspense 与旧版 React。

**useSetAtom / useAtom**

- `useSetAtom` 返回稳定的 `store.set` 回调。
- `useAtom` 组合 `useAtomValue` 与 `useSetAtom`。

## 10. utils 层的实现风格

`src/vanilla/utils/*` 中的大多数工具本质上都是：

- 通过 `atom(read, write)` 组合构建派生行为。
- 通过 `get/set` 协议复用 store 的依赖追踪与调度。

因此工具层不需要自建调度系统，只是对核心能力的“语义化封装”。

## 11. 可插拔内核（Building Blocks）

`internals.ts` 采用 **building blocks** 结构：

- 内部函数（read/write/mount 等）通过数组索引统一调度。
- 支持 `enhanceBuildingBlocks` 或 `INTERNAL_overrideCreateStore` 等方式插入自定义实现。

这为调试工具、实验特性提供扩展入口，但对外仍保持最小 API。

---

如果你希望更深入某一块（例如某个 utils atom 或调度细节），可以告诉我具体文件或关注点，我可以继续细化。
