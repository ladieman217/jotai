# Jotai 仓库实现原理深度文档（Codex 5.3 High）

## 0. 文档目标与阅读范围

这份文档的目标不是介绍“怎么用 Jotai”，而是系统解释这个仓库在工程上是如何实现“原子化状态管理”的，重点覆盖：

1. 核心运行时（`src/vanilla/*`）如何维护状态、依赖图和版本。
2. 写入后如何做失效传播、拓扑重算、批量通知。
3. 异步原子（Promise）如何和中断（Abort）一起工作。
4. React 适配层（`src/react/*`）如何把 store 接入组件生命周期和 Suspense。
5. 工具层（`src/vanilla/utils/*`、`src/react/utils/*`）的共性设计模式。
6. 构建与发布（Rollup + postbuild patch）如何产出多格式包。
7. 测试如何覆盖回归点、性能语义和内存语义。

文中所有结论都尽量绑定源码路径，便于你自行 trace。

---

## 1. 仓库结构与分层边界

### 1.1 顶层结构

- 核心代码：`src/`
- 测试：`tests/`
- 文档源：`docs/`
- 文档站点（Gatsby）：`website/`
- 构建配置：`rollup.config.mjs`

这个仓库本身是单 package workspace（`pnpm-workspace.yaml` 只包含 `.`），但包含多个产物入口。

### 1.2 入口分层

公开 API 由两层构成：

1. `src/vanilla.ts`
2. `src/react.ts`
3. `src/index.ts` 把两者合并导出

对应关系：

- `src/index.ts`：`export * from './vanilla.ts'` + `export * from './react.ts'`
- `src/vanilla.ts`：导出 `atom`、`createStore`、`getDefaultStore`、类型工具
- `src/react.ts`：导出 `Provider/useStore/useAtomValue/useSetAtom/useAtom`

这意味着 Jotai 核心运行时本质上是“React 无关”的，React 部分只是一层绑定。

---

## 2. 核心抽象：Atom 是配置，值在 Store

### 2.1 Atom 类型模型

`src/vanilla/atom.ts` 定义了三类核心类型：

1. `Atom<Value>`：只读（至少有 `read`）
2. `WritableAtom<Value, Args, Result>`：可写（有 `write`）
3. `PrimitiveAtom<Value>`：可写且 write 行为等价于 setState

关键语义：

- atom 对象本身不存“运行中值”。
- atom 只存定义：`read/write/onMount/debugLabel/...`。
- 值与错误都在 store 的内部 map 中。

文档也明确写了这点：`docs/core/atom.mdx` 中“atom config object doesn't hold a value”。

### 2.2 `atom(...)` 重载与默认行为

`atom` 函数通过参数形态分辨：

1. 只传函数：读派生 atom
2. 传 `read + write`：可写派生 atom
3. 传初始值（可选 write）：primitive 或 write-only 派生

内部实现的关键点：

- 每个 atom 分配全局递增 key（`atom1`、`atom2`...）
- `toString()` 在开发模式会拼接 `debugLabel`
- primitive atom 默认 `read` 是 `get(this)`
- primitive atom 默认 `write` 支持函数更新（`prev => next`）

这使 primitive atom 的开发体验近似 `useState`，但底层仍是图计算系统。

---

## 3. Store 总览：公开 API 极小，内部引擎很大

### 3.1 对外 Store API 只有 3 个动作

`src/vanilla/store.ts` 公开：

1. `createStore()`
2. `getDefaultStore()`
3. `INTERNAL_overrideCreateStore()`（内部/高级扩展）

store 实例只有三方法：

- `get(atom)`
- `set(atom, ...args)`
- `sub(atom, listener)`

### 3.2 真正引擎在 `internals.ts`

`src/vanilla/internals.ts`（1000+ 行）定义了：

1. 状态数据结构
2. 算法 building blocks
3. store 构造逻辑
4. hook/扩展机制
5. Promise 中断机制

`createStore()` 实际调用 `INTERNAL_buildStoreRev2`。

---

## 4. 内部状态数据结构（这是整个系统的根）

### 4.1 `AtomState`

每个 atom 在某个 store 中对应一份 `AtomState`，核心字段：

1. `d: Map<depAtom, depEpoch>`
   - 记录“我依赖了谁”和依赖当时版本号。
2. `p: Set<atom>`
   - 记录“哪些 atom 在等待我这个 pending promise”。
3. `n: EpochNumber`
   - 当前版本号。
4. `v?: Value`
   - 当前值（可能是 Promise）。
5. `e?: Error`
   - 当前错误。

### 4.2 `Mounted`

只有“被订阅或作为被订阅原子的传递依赖”的 atom 才有 mounted state：

1. `l: Set<listener>` 当前订阅者
2. `d: Set<depAtom>` 当前挂载依赖
3. `t: Set<dependentAtom>` 当前挂载反向依赖
4. `u?: onUnmount` 卸载时执行

### 4.3 核心容器

`buildStore` 初始化时创建：

1. `atomStateMap: WeakMap<Atom, AtomState>`
2. `mountedMap: WeakMap<Atom, Mounted>`
3. `invalidatedAtoms: WeakMap<Atom, Epoch>`
4. `changedAtoms: Set<Atom>`
5. `mountCallbacks/unmountCallbacks`

为什么是 WeakMap：

- atom 对象无外部引用时可被 GC。
- state 与 mounted 元数据跟随回收。
- 对应测试：`tests/vanilla/memoryleaks.test.ts`。

---

## 5. “building blocks” 机制：可替换内核拼装

`internals.ts` 用一个长度固定的 tuple（`BuildingBlocks`）存全部运行模块，索引 0..27 分别对应状态、拦截器、算法、store API、abort 支持。

例如：

1. `11` 是 `ensureAtomState`
2. `14` 是 `readAtomState`
3. `16` 是 `writeAtomState`
4. `21/22/23` 是 `storeGet/storeSet/storeSub`
5. `24` 是 `enhanceBuildingBlocks`

这种设计的意义：

1. 各函数通过索引拿当前 store 的 building blocks，避免闭包固化旧引用。
2. 允许内部重组、devtools 扩展、hook 注入。
3. 配合 `INTERNAL_buildStoreRev2` 能局部替换内部实现。

`tests/vanilla/internals.test.tsx` 专门验证了：

- blocks 非 sparse
- 不持有 stale references
- internal/external block 替换行为正确

---

## 6. `ensureAtomState`：惰性创建状态

`BUILDING_BLOCK_ensureAtomState` 逻辑：

1. 从 `atomStateMap` 取现有 state。
2. 无则创建 `{ d: new Map(), p: new Set(), n: 0 }`。
3. 触发 store hook `i`（init）。
4. 调用 atom 的 `INTERNAL_onInit`（若定义）。

这解释了为什么“仅创建 atom 不会有值，首次使用才初始化”。

---

## 7. 读路径（`store.get`）完整拆解

入口：`store.get(atom)` -> `storeGet` -> `readAtomState` -> `returnAtomValue`

### 7.1 缓存判定

`readAtomState` 首先判断：

1. atom 已初始化？
2. 若 atom 已 mounted 且未被 invalidated：直接返回缓存。
3. 若未 mounted，则逐个检查依赖版本（`d` 中记录的 `n`）是否变化。
4. 依赖没变也返回缓存。

### 7.2 必须重算时的流程

重算阶段做了很多关键事：

1. 记录 `prevDeps`，准备运行后 prune 旧依赖。
2. 创建 `nextDeps`。
3. 构造内部 `getter(a)`：
   - `a === self` 时走 init 分支
   - `a !== self` 时递归 `readAtomState`
   - 无论是否异常，都会在 `finally` 里记录依赖边（`atomState.d.set(a, aState.n)`）
4. 若当前 atom value 是 Promise，会把“自己依赖了谁”登记到依赖方 `p`。
5. 若当前 atom 已 mounted，依赖的 mounted 反向边也会更新（`mountedMap.get(a)?.t.add(atom)`）。

### 7.3 `read` 选项 `signal` / `setSelf`

`read(get, options)` 中 options 有两个 getter：

1. `signal`
   - 惰性创建 `AbortController`。
2. `setSelf`
   - 已标注 deprecated。
   - 仅 writable atom 可用。
   - sync 调用会给 warning。

### 7.4 值与错误写回

`atomRead(...)` 返回 `valueOrPromise` 后：

1. `setAtomStateValueOrPromise` 写入 `v`，清理 `e`，必要时 bump `n`
2. Promise 时注册 abort handler
3. Promise resolve/reject 后会触发依赖清理与依赖挂载补偿
4. `storeHooks.r`（read hook）触发

异常分支：

- 删除 `v`，设置 `e`，`n++`

`finally` 阶段：

- 若版本发生变化并且与 invalidated epoch 对得上，则把 atom 记入 `changedAtoms` 并触发 `storeHooks.c`。

这一套保证了：

- read 可缓存
- read 可追踪依赖
- read 与 Promise 并发可协作

---

## 8. 写路径（`store.set`）完整拆解

入口：`store.set(atom, ...args)` -> `storeSet`。

`storeSet` 只做两件事：

1. 调 `writeAtomState`
2. 在 finally 中统一 `recomputeInvalidatedAtoms + flushCallbacks`

### 8.1 `writeAtomState` 内部 getter/setter 语义

`writeAtomState` 为 atom.write 构造：

1. `getter(a)`：不做依赖追踪，直接 `returnAtomValue(readAtomState(a))`
2. `setter(a, ...args)`：
   - 若 `a === 当前 write 目标 atom` 且它有 init（primitive/writable）
     - 直接 `setAtomStateValueOrPromise`
     - `mountDependencies`
     - 版本变更则加入 `changedAtoms`
     - 递归 `invalidateDependents`
   - 若 `a !== 当前 atom`
     - 递归调用 `writeAtomState` 写别的 atom

这解释了 docs 里的说法：

- write 函数里的 `get` 不追踪依赖
- write 里的 `set` 可以触发目标 atom 的 write

### 8.2 同步/异步写完成时机

`writeAtomState` 用 `isSync` 控制：

- 在 write 过程结束后才把 `isSync=false`
- setter finally 中如果是异步阶段才触发重算+flush

目的：

- 避免同步递归写期间重复 flush
- 把一批变更汇总后再统一提交

---

## 9. 失效传播与拓扑重算

### 9.1 `invalidateDependents`

从某 atom 出发，沿“挂载依赖 + pending 依赖”图向上 DFS：

1. 对每个 dependent 写入 `invalidatedAtoms.set(dep, depState.n)`
2. 避免重复入栈

注意它依赖 `getMountedOrPendingDependents`，所以 pending promise 链路也会被纳入。

### 9.2 `recomputeInvalidatedAtoms`

这是算法核心，分两步：

1. 从 `changedAtoms` 出发做 DFS，生成逆拓扑序列表。
2. 逆序回放：
   - 检查该 atom 的依赖中是否有 changed 的
   - 有才重读（`readAtomState`）并刷新挂载依赖（`mountDependencies`）
   - 删除其 invalidated 标记

源码注释明确提到：为了简化和性能，不显式做 cycle 检测。

`tests/vanilla/internals.test.tsx` 有案例保证不会多路径重复 invalidation。

---

## 10. 回调批处理 `flushCallbacks`

`flushCallbacks` 是“提交阶段”：

1. 收集 `changedAtoms` 对应 mounted listeners。
2. 合并 mount/unmount 回调。
3. 执行回调并捕获错误。
4. 若执行回调期间又产生 changedAtoms，则再次 `recomputeInvalidatedAtoms`。
5. 循环直到 changed/mount/unmount 全为空。
6. 有错误则抛 `AggregateError`。

这保证了：

- listener 感知到的是稳定状态
- 连锁写入不会造成半更新状态泄露

---

## 11. 挂载系统（mount/unmount）

### 11.1 `store.sub` 的实际行为

`store.sub(atom, listener)`：

1. `mountAtom(atom)`
2. `mounted.l.add(listener)`
3. 立即 `flushCallbacks`
4. 返回 `unsub`

`unsub`：

1. 删除 listener
2. `unmountAtom(atom)`
3. `flushCallbacks`

### 11.2 `mountAtom`

首次 mount 时：

1. 先 `readAtomState(atom)`，保证依赖关系最新
2. 先 mount 所有依赖，并把自己登记到依赖的 `t`
3. 创建自己的 `Mounted { l,d,t }`
4. writable atom 会把 `onMount` 调度进 `mountCallbacks`

`onMount` 里注入的 `setAtom` 也遵守“异步后统一 flush”的约束。

### 11.3 `unmountAtom`

卸载条件：

1. 自己没有 listener
2. 没有仍有效的 dependent 需要它

满足后：

1. 记录 `onUnmount` 到 unmountCallbacks
2. 从 mountedMap 删除自己
3. 递归卸载依赖（并从依赖 `t` 中删除自己）

所以 Jotai 的“生命周期”是依赖图驱动而不是组件树驱动。

---

## 12. Promise 与 Abort 机制

### 12.1 Promise 是 atom 值的一等形式

`setAtomStateValueOrPromise` 不区分 sync/async：

1. 统一写到 `atomState.v`
2. 值变化则 `n++`
3. 上一个值若是 Promise，会调用 `abortPromise` 触发已注册中断

### 12.2 abort handlers

内部有 `abortHandlersMap: WeakMap<promise, Set<handler>>`：

1. `registerAbortHandler` 给某 promise 绑定中断行为
2. promise settle 后自动清理 map
3. `abortPromise` 会执行所有 handler

read 路径会把 `controller.abort()` 绑定进去，因此新一轮计算可以中断旧的异步任务。

### 12.3 React 侧的 continuable promise

`useAtomValue` 使用 `createContinuablePromise` 解决一个实战问题：

- UI 正在等 Promise A
- A 被中断后切到 Promise B
- 不希望 Suspense 边界重复出现不必要抖动

做法：

1. 通过 store 内部 `registerAbortHandler` 监听 promise 切换
2. 中断时调用 `getValue()` 获取新值
3. 若新值还是 promise，则把同一个 continuable promise 继续链到新 promise
4. 最终对组件暴露一个“可继续”的 promise

这部分是 v2 异步体验的关键工程化补丁。

---

## 13. React 适配层细节

### 13.1 Provider / useStore

`useStore(options)` 优先级：

1. `options.store`
2. Context store
3. `getDefaultStore()`

`Provider` 若未传 store，会在组件内 `useRef` 惰性创建一个 store。

### 13.2 `useAtomValue`

内部用 `useReducer` 保存三元组：

1. 当前值
2. 当前 store 引用
3. 当前 atom 引用

只要三者 `Object.is`/引用都不变就复用旧状态，避免无谓 render。

订阅逻辑：

1. `store.sub(atom, callback)`
2. callback 内 rerender
3. 可选 `delay` 延迟 rerender
4. 可选 `unstable_promiseStatus` 给 promise 打状态标签

当值是 Promise 时：

- 使用 `use(promise)`（或 shim）抛给 Suspense

### 13.3 `useSetAtom` / `useAtom`

- `useSetAtom` 只是 `useCallback(() => store.set(...))`
- dev 模式对“只读 atom 被 set”会抛错
- `useAtom` 就是 `useAtomValue + useSetAtom` 的组合

---

## 14. Store hooks（实验性观测点）

`internals.ts` 提供 `StoreHooks`：

1. `i` init
2. `r` read
3. `c` change
4. `m` mount
5. `u` unmount
6. `f` flush

`INTERNAL_initializeStoreHooksRev2` 会把未定义 hook 填成可订阅回调。

`tests/vanilla/internals.test.tsx` 对每个 hook 都有行为测试。

这也是 devtools/调试扩展能够接入内核的基础。

---

## 15. 默认 store 与多实例风险

`getDefaultStore()` 是 provider-less 模式入口。源码有一个开发时防护：

- 把 default store 挂到 `globalThis.__JOTAI_DEFAULT_STORE__`
- 检测到多个 Jotai 实例时发 warning

目的：

- 避免多份包实例导致“你以为在同一个默认 store，实际不是”的隐蔽 bug。

---

## 16. Vanilla utils 设计范式

很多 utils 共享三种模式：

1. **高阶 atom 包装**：用少量 private atom 拼装功能
2. **WeakMap memo**：保证同参数返回稳定 atom 实例
3. **运行时兼容 sync/async**：尽可能把 Promise 当正常值处理

下面逐个拆关键点。

### 16.1 `atomWithReset`

- 支持 `RESET` sentinel 重置到初值。
- write 支持函数更新。

### 16.2 `atomWithReducer`

- 把 reducer 写法包装成 primitive atom write。
- 实现极薄，核心还是 `set(this, reducer(get(this), action))`。

### 16.3 `atomWithDefault`

- 用 `overwrittenAtom` 记录是否覆盖默认值。
- 若未覆盖则动态执行 `getDefault(get, options)`。

### 16.4 `atomWithRefresh`

- 用私有 `refreshAtom` 计数触发重读。
- 调 write 且无参数时等价“强制刷新”。

### 16.5 `atomWithLazy`

- 通过 `Object.defineProperty(a, 'init', getter)` 延迟初值计算。
- 每个 store 首次使用时才求值。

### 16.6 `selectAtom`

- 三层 WeakMap memo（atom + selector + equalityFn）。
- 通过 `prevSlice` + `equalityFn` 保持 slice 稳定引用。

### 16.7 `splitAtom`

- 把数组 atom 拆成 item atom 列表。
- 用 keyExtractor 保持 item atom 稳定（重排也尽量复用）。
- 支持 `remove/insert/move` action。
- 包含越界容错分支（读时可返回 stale 值以兼容某些动画场景）。

### 16.8 `unwrap`

- 把“值可能是 Promise”的 atom 转成“同步可读 + fallback”接口。
- 用两个 WeakMap 缓存 promise 的结果和错误。
- 通过内部 refresh atom 在 promise settle 后触发重算。

### 16.9 `atomWithStorage`

- 提供 sync/async storage 抽象。
- `createJSONStorage` 统一 JSON parse/stringify + 可选 reviver/replacer。
- onMount 时主动从 storage 拉值并订阅外部变更（含浏览器 `storage` 事件 fallback）。
- 支持 `RESET` 删除持久化键。

### 16.10 `atomWithObservable`

- 接收 Observable 或 Subject。
- 若没 `initialValue`，初次值可为 pending Promise。
- onMount/onUnmount 管理订阅生命周期。
- writable 分支仅在 observable 有 `next`（Subject）时可写。

### 16.11 `freezeAtom`

- read 结果 deepFreeze。
- write 到自身时也 deepFreeze 新值。
- 用 WeakSet 防止重复包裹。

### 16.12 `atomFamily` / `loadable` 等迁移态 API

仓库当前版本里这类 util 带明确 deprecation warning，迁往独立包或 userland recipe：

- `atomFamily` -> `jotai-family`
- `loadable` 倾向 `unwrap` 自己组合

这反映出仓库的维护策略：核心尽量瘦，生态能力外移。

---

## 17. React utils 实现要点

### 17.1 `useHydrateAtoms`

- 用 `WeakMap<Store, WeakSet<Atom>>` 记录“该 store 已 hydrate 过哪些 atom”。
- 默认同一个 atom 在同一 store 只 hydrate 一次。
- `dangerouslyForceHydrate` 可强制重复 hydrate。

### 17.2 `useAtomCallback`

- `useMemo` 动态创建 write-only atom，把 callback 包进 write。
- 返回 `useSetAtom`，可在组件外观上表现成可调用函数。

### 17.3 `useResetAtom`

- 本质是 `set(RESET)` 的包装。

### 17.4 `useReducerAtom`（deprecated）

- 只是 `useAtom + reducer` dispatch 包装。
- 已引导用户按 recipe 自建。

---

## 18. Babel 插件层（开发体验增强）

`src/babel/*` 提供三项能力（当前都 deprecated，迁移到 `jotai-babel`）：

1. `plugin-debug-label`
2. `plugin-react-refresh`
3. `preset`

### 18.1 `plugin-debug-label`

- 识别 atom 创建调用（`atom/atomFamily/...` 以及 custom names）。
- 给变量 atom 自动追加 `atom.debugLabel = 'variableName'`。
- 处理 default export atom 时会重写导出结构。

### 18.2 `plugin-react-refresh`

- 在 `globalThis` 上挂 `jotaiAtomCache`（Map）。
- 用 `filename + exportName` 当 key 缓存 atom 实例。
- HMR 时尽量保持 atom identity 稳定，减少状态丢失。

### 18.3 `babel/utils.ts` 的识别名单

内置了较多生态函数名（例如 `focusAtom`、`atomWithMachine`、`atomWithStore` 等），反映了 Jotai 生态横向整合方式。

---

## 19. 构建与发布流水线

### 19.1 Rollup 多目标产物

`rollup.config.mjs` 为每个入口构建：

1. CJS
2. ESM
3. UMD（dev/prod）
4. System（dev/prod）
5. d.ts（部分入口）

### 19.2 `import.meta.env?.MODE` 处理

不同目标会把 `import.meta.env?.MODE` 替换为：

1. CJS：`process.env.NODE_ENV`
2. ESM：兼容表达式
3. UMD/System：固定字符串（`development`/`production`）

这让源码能统一使用同一判断分支。

### 19.3 postbuild patch

`package.json` 里的 postbuild 链路做了大量兼容处理：

1. patch d.ts import 路径
2. 复制 dist 结构
3. 生成 ts3.8 降级类型
4. patch esm d.ts 扩展名为 `.mts`
5. 清理 dist package.json 字段

这部分体现了库作者对“历史 TS/工具链兼容”的工程投入。

### 19.4 exports / typesVersions

`package.json` 通过 `exports` 区分：

1. react-native
2. import (esm)
3. default (cjs)

并用 `typesVersions` 兼容不同 TS 版本。

---

## 20. 测试体系与回归策略

### 20.1 测试配置

`vitest.config.mts` 关键点：

1. alias `jotai` 到 `src`，测试直接打源码
2. jsdom 环境
3. React plugin 可挂 debug-label 插件（若 dist 存在）

### 20.2 测试分层

- `tests/vanilla/*`: store/依赖/并发/内存
- `tests/react/*`: hooks/渲染/StrictMode/Suspense
- `tests/babel/*`: 插件 transform

### 20.3 高价值回归点示例

1. **内存泄漏**：`memoryleaks.test.ts` 验证 WeakMap 设计目标。
2. **异步链传播**：`dependency.test.tsx` 覆盖 async chain、条件依赖、never-resolving promise。
3. **优化语义**：`react/optimization.test.tsx` 验证“只相关组件重渲染”。
4. **中断语义**：`react/abortable.test.tsx` 验证 `signal.aborted`、abort event listener、切换场景。
5. **内部扩展点**：`internals.test.tsx`、`storedev.test.tsx` 验证 building blocks/hook/devstore 行为。

---

## 21. 文档站点实现（repo 内第二系统）

`website/` 是 Gatsby + MDX 文档站，不是核心 runtime 但决定了知识分发路径。

主要特点：

1. Gatsby 构建
2. MDX 文档渲染
3. docs 源文件在仓库根 `docs/`

这保证“代码与文档共仓、同版本演进”。

---

## 22. 端到端执行时序（四条完整链路）

### 22.1 时序 A：primitive atom 同步更新

场景：`countAtom = atom(0)`，组件通过 `useAtom(countAtom)` 订阅并点击按钮 `setCount(c=>c+1)`。

执行链：

1. React 事件触发 `useSetAtom` 返回函数。
2. 调用 `store.set(countAtom, updater)`。
3. `writeAtomState` 命中 `a===atom` 分支，计算新值写入 `atomState.v`。
4. `Object.is` 不同，`n++`，加入 `changedAtoms`。
5. `invalidateDependents(countAtom)` 标记依赖它的派生 atom。
6. `storeSet finally` 进入 `recomputeInvalidatedAtoms`。
7. 受影响派生 atom 重算并刷新 mounted 依赖边。
8. `flushCallbacks` 收集 `countAtom` 与受影响 atom 的 listeners。
9. React 组件 rerender，`useAtomValue` 的 reducer 比较新旧值后提交。

关键性质：

- 只通知订阅到相关 atom 的组件。
- 同值写入不会 bump epoch，不触发订阅。

### 22.2 时序 B：条件依赖切换

场景：

- `a = atom(...)`
- `b = atom(...)`
- `c = atom(get => flag ? get(a) : get(b))`

当 flag 从 true 变 false：

1. `c` 重算时 `getter` 只访问了 `b`。
2. `nextDeps` 记录 `b`。
3. 重算完成执行 `pruneDependencies`，把 `a` 从 `c.d` 中删除。
4. `mountDependencies` 同步 mounted 边：移除 `a -> c`，建立 `b -> c`。
5. 后续 `a` 更新不会再触发 `c`。

这正是 docs `use-atom.mdx` 所说“每次 read 刷新依赖图”的具体实现。

### 22.3 时序 C：异步 atom + 快速切换 + abort

场景：`userAtom = atom(async (get, {signal}) => fetch(url, {signal}))`，url 连续变化。

1. 第一次读得到 Promise P1，注册 abort handler。
2. url 改变导致重算，生成 Promise P2。
3. `setAtomStateValueOrPromise` 发现旧值是 promise，调用 `abortPromise(P1)`。
4. P1 对应 handler 执行 `controller.abort()`。
5. UI 层 `useAtomValue` 上，continuable promise 会把等待链从 P1 迁移到 P2。
6. 最终组件拿到 P2 结果，避免显示已过期结果。

### 22.4 时序 D：订阅卸载与依赖回收

1. `store.sub(derivedAtom, listener)` 会挂载 derived 及其依赖链。
2. `unsub()` 删除 listener。
3. `unmountAtom` 检查是否仍被其他 mounted atom 依赖。
4. 若无，卸载自己并递归卸载独占依赖。
5. `onUnmount` 回调通过 unmountCallbacks 在 flush 阶段执行。

结果：

- 依赖子图可回收。
- 无活跃订阅时不会持续计算无关派生。

---

## 23. 性能语义与工程取舍

### 23.1 为什么它通常“重渲染少”

1. atom 粒度天然细。
2. version + dependency map 做局部缓存命中。
3. invalidated + topo recompute 只重算受影响子图。
4. callback flush 批处理减少重复通知。

### 23.2 明确可见的取舍

1. **不做显式循环依赖检测**
   - 代码注释中明确提到简化与性能考虑。
2. **内部索引 tuple 可读性换可替换性**
   - 需要配合类型保证和测试保证。
3. **异步一致性优先于简单实现**
   - continuable promise + abort handler 复杂度较高，但大幅减少并发边界问题。

### 23.3 对使用者的隐含要求

1. atom identity 必须稳定（render 内创建要 `useMemo/useRef`）。
2. write 中尽量避免把重计算塞进 render 读路径。
3. SSR 场景应使用 Provider 隔离请求级 store（避免默认全局 store 跨请求共享）。

---

## 24. 版本迁移信号与生态策略

从源码中可见大量 deprecation 提示，方向一致：

1. 核心包保持精简。
2. 某些能力迁出到独立包（如 `jotai-family`、`jotai-babel`）。
3. 推荐 userland recipe 替代内置“可组合就不内建”的工具。

这是一种“内核稳定、边缘外移”的演进策略。

---

## 25. 结论：Jotai 内核的本质

如果把这套实现压缩成一句工程定义：

**Jotai 是一个以 atom 对象为节点、以 read 依赖为边、以 epoch 为一致性标记、以拓扑回放为更新策略、以 WeakMap 为生命周期容器、以 Promise/Abort 为并发协议的增量状态图引擎；React 仅是它的订阅与渲染适配层。**

这句话对应你在源码里看到的每一个关键模块：

1. 节点定义：`src/vanilla/atom.ts`
2. 图状态与算法：`src/vanilla/internals.ts`
3. store API：`src/vanilla/store.ts`
4. React bridge：`src/react/*`
5. 生态工具：`src/vanilla/utils/*`, `src/react/utils/*`
6. 工程支撑：`rollup.config.mjs`, `tests/*`, `docs/*`

---

## 26. 附录：快速索引（按你最常回看的问题）

### Q1. 为什么写一个 atom 会导致另一个 atom 更新？

看：

1. `invalidateDependents`
2. `recomputeInvalidatedAtoms`
3. `mountDependencies`

都在 `src/vanilla/internals.ts`。

### Q2. 为什么某些 set 不触发重渲染？

看：

1. `setAtomStateValueOrPromise` 的 `Object.is` 比较
2. `useAtomValue` reducer 的 `Object.is` + store/atom 引用比较

### Q3. 为什么 async 读不会总是产生竞态脏数据？

看：

1. `registerAbortHandler/abortPromise`
2. read options 里的 `signal`
3. React 侧 `createContinuablePromise`

### Q4. 怎么做内核观测和调试扩展？

看：

1. `StoreHooks`（`i/r/c/m/u/f`）
2. `INTERNAL_buildStoreRev2` + blocks 替换
3. `tests/vanilla/storedev.test.tsx`

---

## 27. 参考源码路径清单

- `src/index.ts`
- `src/vanilla.ts`
- `src/react.ts`
- `src/vanilla/atom.ts`
- `src/vanilla/store.ts`
- `src/vanilla/internals.ts`
- `src/react/Provider.ts`
- `src/react/useAtomValue.ts`
- `src/react/useSetAtom.ts`
- `src/react/useAtom.ts`
- `src/vanilla/utils/*.ts`
- `src/react/utils/*.ts`
- `src/babel/*.ts`
- `rollup.config.mjs`
- `package.json`
- `vitest.config.mts`
- `tests/vanilla/*`
- `tests/react/*`
- `tests/babel/*`
- `docs/core/*.mdx`
- `docs/guides/core-internals.mdx`

