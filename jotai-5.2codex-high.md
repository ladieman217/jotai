# 当前仓库（Jotai）实现原理概述

> 说明：以下内容基于当前仓库源码的实现方式进行总结，重点覆盖核心数据结构、依赖追踪、更新调度与 React 集成路径。

## 1. 代码结构总览（核心路径）

- `src/vanilla/atom.ts`：定义 atom 工厂与基础读写协议（`read`/`write`）。
- `src/vanilla/internals.ts`：Store 的内部实现、依赖图、更新调度与异步处理核心。
- `src/vanilla/store.ts`：对外暴露 `createStore` / `getDefaultStore`，并连接内部实现。
- `src/react/*`：React 集成（Provider、useAtomValue、useSetAtom、useAtom）。

## 2. Atom 模型：最小状态与统一读写协议

- `atom(...)` 工厂会为每个 atom 生成唯一 key，并创建包含 `read`/`write` 的配置对象。
- **基础 atom**：若传入初始值，会以默认 `read`/`write` 读写自身值。
- **派生 atom**：`read` 在运行时通过 `get` 读取其他 atom，形成依赖关系；可选 `write` 承担更新逻辑。

## 3. Store 的核心状态结构（internals.ts）

- **AtomState（每个 atom 一个）**：
  - `d: Map<atom, epoch>`：依赖关系 + 依赖的版本号（epoch）。
  - `p: Set<atom>`：依赖该 atom 且处于 pending promise 的 atom 集合。
  - `n: epoch`：该 atom 当前版本号。
  - `v` / `e`：当前值或错误。
- **Mounted（仅挂载 atom 才有）**：
  - `l`：订阅监听集合。
  - `d`：当前挂载状态下的依赖集合。
  - `t`：依赖当前 atom 的“上游”集合。
  - `u`：卸载回调（onMount 的返回）。

这些结构通过 `WeakMap` 管理，便于在无人引用时回收。

## 4. 读取流程：依赖追踪与缓存复用

读取入口是 `readAtomState`：

- **缓存判断**：
  - 已初始化且未失效（invalidated） → 直接返回。
  - 依赖的 epoch 全部未变化 → 直接返回。
- **重新计算**：
  - 清空旧依赖集合 `d`，运行 `atom.read(get, options)`。
  - 在 `getter` 中记录依赖：把被读取 atom 写入 `d`，并记录其 epoch。
  - 若读取到 Promise，会注册 abort handler，并将 pending 关系加入 `p` 集合。
  - 将结果写回 AtomState（`setAtomStateValueOrPromise`）并递增 epoch。

依赖图因此在每次读取时被增量维护。

## 5. 写入流程：局部更新与失效传播

写入入口是 `writeAtomState` / `store.set`：

- **写自身（基础 atom）**：
  - 直接更新值，递增 epoch。
  - 标记为 changed，并调用 `invalidateDependents` 使依赖它的 atom 进入“失效”状态。
- **写派生 atom**：
  - 执行自定义 `atom.write(get, set, ...args)`，内部可批量更新多个 atom。
  - 完成后会触发 `recomputeInvalidatedAtoms` + `flushCallbacks`。

写入整体遵循“只影响依赖链”的原则。

## 6. 失效与重新计算调度

- `invalidateDependents` 会沿依赖图向上标记 invalidated。
- `recomputeInvalidatedAtoms` 会以 **拓扑顺序** 遍历 changed atoms 的依赖链：
  - 只在依赖变化时重算。
  - 重算后重新挂载依赖并更新依赖图。

这样可以做到最小范围的重新计算。

## 7. 挂载/订阅生命周期

- `store.sub(atom, listener)`：
  - `mountAtom`：确保 atom 自身与依赖全部挂载。
  - 将 listener 加入 `mounted.l`。
- `unmountAtom`：
  - 当 atom 没有订阅且不再被依赖时，触发 onUnmount，并递归卸载其依赖。

这使得依赖图保持在“使用中”的最小集合。

## 8. 异步与 Promise 处理

内部专门维护 `promiseStateMap`：

- 当 atom 的值是 Promise：
  - 注册 abort handler（依赖变化时可中止旧 Promise）。
  - 依赖关系会被加入到 `p` 集合，确保变更能传递到等待中的 atom。
- 若新值替换旧 Promise，会主动 abort 旧 Promise。

## 9. React 集成（src/react）

- `Provider` 用 Context 注入 store，可使用外部 store 或内部 `createStore`。
- `useAtomValue`：
  - 通过 `useReducer` 缓存 [value, store, atom]，避免无效 rerender。
  - 订阅 `store.sub`，在变更时触发 rerender。
  - 对 Promise 值做 “continuable Promise” 包装，适配 Suspense 或旧版 React。
- `useSetAtom`：返回 `store.set` 的稳定回调。
- `useAtom`：组合 `useAtomValue` + `useSetAtom`。

## 10. 扩展与调试钩子

`StoreHooks` 提供创建、读取、变更、挂载、卸载、flush 的订阅入口，便于调试或扩展工具链。

---

如果你想要更深入的细节（比如某个工具函数、某个 utils atom 的实现路径），告诉我具体文件或关注点即可继续细化。
