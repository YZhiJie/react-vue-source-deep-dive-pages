# 漫画 · 第 15 话 组件的一生：从 setup 到 DOM

> 对应正文：[第 5 章 组件系统：从 setup() 到真实 DOM 的完整生命周期](../../docs/vue/05-component-system.md) ｜ 原画：[05-component-system.svg](./05-component-system.svg)

## 登场角色

- **阿V**：绿发少女主角，这一话本色出演一个组件，亲自走一遍「从 VNode 到 DOM」的完整一生。
- **阿R**：蓝色机器人，从 React 餐厅串门过来，盯着那个 ReactiveEffect 罩子若有所思。
- **码叔**：猫头鹰教授，片尾照例飞出，三句话把挂载、副作用、属性查找收成口诀。

## 剧情梗概

阿R 总听人说组件「挂载」「更新」，却不知道这些词在框架内部到底对应什么。阿V 决定亲自示范：她从一张组件 VNode 出发，被 new 成内部实例，跑完 setup()，被 render 函数描述成子树，最后由 patch 端上真实 DOM。

上桌之后她也没闲着：一个 ReactiveEffect 罩子把「render 加 patch 子树」整段罩住，数据一变，罩子顶灯亮起，她就重跑一遍。阿R 又指着模板里的 `{{ msg }}` 问这 msg 从哪儿来，阿V 带他按顺序连敲三扇门：setupState、props、data，门旁还挂着一块 accessCache 加速牌。

## 分格解读

### 第 1 格 · 组件的一生

时间线五站对应源码里的真实接力：**组件 VNode → new 实例 → setup() → render 函数 → patch 上桌**。

父组件 render 遇到组件标签产出组件 VNode，`patch` 看到 `ShapeFlags.STATEFUL_COMPONENT` 就分派给 `mountComponent`：先 `createComponentInstance` new 出 `ComponentInternalInstance`（uid、EffectScope、provides 全在此刻就位），再 `setupComponent` 初始化 props/slots 并执行 setup()，拿到 setup 返回的响应式状态（对象会经 proxyRefs 挂到 setupState）。

随后 `finishComponentSetup` 补齐 render 函数（setup 返回函数、SFC 预编译、运行时编译模板三选一），effect 首跑时 render 产出 subTree，由 patch 挂上真实餐桌。阿V 脚下每一站，都能在 component.ts 与 renderer.ts 里找到同名函数。

顺带一提，初始化 props 时挂的是 shallowReactive：替换整个 prop 引用能被追踪，组件内部深改 prop 属性不会触发更新——props 被定位成父组件给的只读快照。

### 第 2 格 · 渲染副作用

罩子就是 `setupRenderEffect` 里 `new ReactiveEffect(componentUpdateFn)` 建的**渲染副作用**：首跑时执行 render 生成 subTree 并 patch 成 DOM，期间读到的 props、setupState 全部被它罩住（track 收集依赖）。

数据一变并不直接重渲染，而是 trigger 走 effect 的 scheduler，把 job 推进 queueJob 调度队列，下一个微任务统一 flush——罩子亮灯，`componentUpdateFn` 走更新分支重跑一次。所以牌上写：**组件更新 = effect 重跑**。同一 tick 里连改三次，灯也只亮一轮，只渲染一遍。

实例上由此分成三个入口：`effect` 是副作用本体，`update` 无条件重跑（首挂载、父组件更新、\$forceUpdate 走它），`job` 带 runIfDirty 脏检查、专供调度队列。

### 第 3 格 · msg 找谁要

模板编译产物里写的是 `_ctx.msg`，`_ctx` 就是 `instance.proxy`。同一个 msg 可能来自 setup 返回值、props 或 Options 的 data，编译器无法预知，运行时就靠 PublicInstanceProxyHandlers 按顺序敲门：**setupState → props → data**（再往后还会问 ctx 与全局属性）。

门旁的小牌是 accessCache：第一次访问要依次 hasOwn 探测，并记下「这把钥匙开哪扇门」；之后直接按缓存的访问类型下标直取，渲染最热路径上零 hasOwn。写入也有路由：setupState 可写就写 setupState，直接改 props 则会被拒绝并警告。

至于 `$el`、`$attrs` 这类美元符号开头的键不进三扇门，它们走专门的 publicPropertiesMap getter 表。

## 码叔划重点

1. 挂载组件 VNode 时创建 ComponentInternalInstance，初始化 props/slots 并执行 setup()，拿到响应式状态与 render。
2. setupRenderEffect 用 ReactiveEffect 把 render + patch 子树罩成渲染副作用，依赖一变 scheduler 入队，effect 重跑即重渲染。
3. 模板变量经 PublicInstanceProxyHandlers 按 setupState/props/data 顺序查找，结果记入 accessCache，后续访问 O(1) 直达。

## 自测一题

**问：** 同一个 tick 里把 `msg.value` 连续改三次，组件会重渲染几遍？为什么？

**答：** 只重渲染一遍。每次 set 都会 trigger 渲染 effect 的 scheduler，但 scheduler 并不直接执行更新，而是把同一个 job 经 queueJob 去重入队；等同步代码跑完、下一个微任务 flush 时由 `runIfDirty` 统一执行一次，`componentUpdateFn` 走更新分支拿到最终值再 patch。多次修改因此被合并成一次渲染。
