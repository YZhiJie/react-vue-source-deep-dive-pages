# 漫画 · 第 12 话 Redux：中央账本与柜员管线

> 对应正文：[第 12 章 Redux 状态管理](../../docs/react/12-redux.md) ｜ 原画：[12-redux.svg](./12-redux.svg)

## 登场角色

- **阿R**：React 餐厅主厨，本话升任连锁餐厅中央账房总管，推行「中央账本 + 柜员管线」制度。
- **阿V**：隔壁 Vue 绿餐厅主厨兼老板娘，看阿R 的分厅越开越多、账本越记越乱，特来取经。
- **码叔**：猫头鹰教授，每话结尾出来划重点。

## 剧情梗概

阿R 的餐厅连锁经营后，各分厅各记各的账，乱成一锅粥。阿R 祭出改革：全集团只设**一本中央账本**（单一 store），谁也不许直接动笔——想改账，先填 **action 申请单**，交给柜员 **reducer** 按固定规则誊写新账页。单据到账房之前，还要过风控、审计等**中间件关卡**，一层层洋葱式放行；拿着「代办委托书」（函数 action）的，thunk 专窗当场代办。最后阿R 用 `useSelector` 栏目订阅和 Immer 抄稿机，回答了阿V「天天重写账本会不会逼疯看账人」的疑问。

## 分格解读

### 第 1 格 · 乱账收归中央

各分厅的小账本东倒西歪，阿V 问改账流程，阿R 立规矩：账只能看、不能改，唯一通道是 `dispatch(action)`。申请单写明「业务类型 type + 附件 payload」，柜员 reducer 是台纯函数机器——凭当前账页和单据，誊写一本**新账**，绝不临场打电话（无副作用）、绝不在旧账上涂改（不可变更新）。

画面里 `action 单` 飞向「中央账本 store」，对应源码：`currentState` 锁在 `createStore` 闭包里，外部摸不到；改它的唯一动作是 dispatch 里那一次同步的 `reducer(state, action)`，执行期间还有 `isDispatching` 锁防止有人递新单子。

### 第 2 格 · 柜员管线三道关

一张「异地调货单」要连过三关：① 风控 m1 → ② 审计 m2 → ③ 账房 reducer，全部放行才誊出新账页。每个中间件是 `store => next => action => {}` 三层柯里化，`compose` 把它们串成右侧的洋葱切面——m1 最先看到单据进站，最后看到结果出站；每关都能拦截、改写、调用 `next(action)` 放行。

旁边紫色的 thunk 专窗是灵魂细节：`fn()` 函数单被认出「这不是单据是代办委托」，当场执行并注入 `dispatch / getState`，reducer 永远见不到函数。所谓异步 action，只是代办人拿着窗口钥匙多跑了几趟同步 dispatch。

### 第 3 格 · 订栏目与抄稿机

阿V 担心账本天天重写、看账人崩溃。阿R 亮出两件法宝：一是**栏目订阅表**——`useSelector` 让每个分厅只订自己那栏，store 摇铃时各栏做引用比对（默认 `===`），栏里数据没变就返回旧引用，组件连重渲染都不触发；二是 **Immer Proxy 抄稿机**——`createSlice` 里柜员可以像在草稿上直接涂改（`state.list.push(x)`），Proxy 暗中记录每笔修改，finalization 时沿修改路径浅拷贝产出新账页，没碰过的分支保持原引用（structural sharing），写着可变、产出的永远是不可变新账。

## 码叔划重点

1. 单一数据源靠闭包物理隔绝：state 只活在 `createStore` 内部，改它的唯一通道是 dispatch 里那一次同步的 `reducer(state, action)`。
2. 中间件是洋葱模型：`compose(...chain)(store.dispatch)` 从右往左串管线；`next` 前进一站，`middlewareAPI.dispatch` 重进整条管线；thunk 核心仅 14 行，专拦函数 action。
3. 精准订阅 + 不可变产出：`useSelector` 引用比对决定组件是否重渲染；`createSlice` 的可变写法由 Immer 的 Proxy draft 与结构共享兜底。

## 自测一题

**问：** 柜员 reducer 图省事，直接在旧账页上涂改后原样交回（`state.list.push(x); return state`），会发生什么？

**答：** 连锁反应全线崩溃：reducer 返回的是同一引用 → `combineReducers` 的 `hasChanged` 引用比较为 false → state 根引用不变 → `useSelector` 的 `Object.is` 比对短路 → **UI 完全不更新，而且没有任何报错**。这正是「纯函数 + 不可变更新」不是风格偏好而是硬契约的原因；用 RTK 让 Immer 接管 draft，可以从根上避免这类静默故障。

## 进阶追问

既然每个 action 都会过所有 slice reducer（`combination` 里没有路由表），大应用里每次 dispatch 岂不是全树遍历？`combineReducers` 靠什么保证这种做法依然便宜？提示：想想 `hasChanged` 的 `!==` 比较和「全都没变就原样返回旧 state」这条退路，再连同 reselect 的两段缓存一起算总账。

## 下一话预告

账本制度好是好，就是太隆重：action type、reducer、slice、thunk、Provider 一样不能少，小本生意也要全套流程。下一话（EP.21 · Zustand）阿V 掏出她冰箱贴的公告栏——没有申请单、没有柜员、没有 Provider，`set` 一笔就改、组件照样精确订阅。原来状态管理，还能这么轻。
