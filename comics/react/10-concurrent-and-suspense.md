# 漫画 · 第 10 话 并发与 Suspense：能放下的半成品

> 对应正文：[第 10 章 并发特性与 Suspense](../../docs/react/10-concurrent-and-suspense.md) ｜ 原画：[10-concurrent-and-suspense.svg](./10-concurrent-and-suspense.svg)

## 登场角色

- **阿R**：React 餐厅主厨，蓝色小机器人，本话亮出「同一双手随时放下半成品」的并发绝技。
- **阿V**：隔壁 Vue 绿餐厅主厨兼老板娘，绿发少女，揣着「并发就是同时炒两锅」的误会来串门。
- **码叔**：猫头鹰教授，每话结尾出来划重点。

## 剧情梗概

阿V 跑进厨房问：「并发就是同时炒两锅？」阿R 连连摆手——JavaScript 只有单线程，所谓并发其实是**协作式调度**：同一双手 render 到一半，能随时放下手里的半成品去接急单。

话音未落，一盘低优先级的 transition 半成品就被「急单！新输入」打断。阿R 不补锅、不将就，整盘倒进回收桶，端起新 state 从头再来。最后是 Suspense 边界的前厅表演：promise 的主菜没好，先给客人上 fallback 零食，菜好的一瞬间无缝换上，阿V 连停顿都没察觉。

## 分格解读

### 第 1 格 · 不是三头六臂

阿V 把并发（concurrent）误当成了并行（parallel）。React 19 仍然跑在单线程上，没有任何真并行计算；阿R 的答案点破本质：协作式调度让 render 阶段可以随时让出主线程，做到**可中断、可丢弃、可重放**。

被放下的只是内存里的 fiber 半成品，commit 还没发生，所以客人永远看不到半盘菜。第 1 话那条「render 纯计算、commit 不可打断」的流水线，正是本话一切魔法的地基。

这也解释了并发的收益从哪来：不是「算得更快」，而是点击、输入等紧急交互永远不必排在大渲染后面干等——响应性来自让位，而不是提速。

### 第 2 格 · 半成品扔掉重做

低优先级的 transition 更新在后台 render 到一半，高优先级的 SyncLane 急单（点击、输入）插队。React 不修补旧半成品，而是**整盘丢弃**：紧急更新提交之后，基于最新 state 重新申请一次渲染。正因为 render 是纯计算，重放才没有副作用代价。

lane 模型为此备了 14 条独立的 TransitionLane，多个 transition 并发推进时互不堵塞；让步也不等于饿死，transition 挂起超过 5 秒会被标记过期、强制同步跑完，页面最终一定收敛。

`useTransition` 的 `isPending` 只反映「这次渲染是否已提交」；对已有内容做刷新时，React 默认继续显示旧数据而不是闪一遍骨架屏，别把它当成普通 loading 布尔值乱用。

### 第 3 格 · 等菜先上零食

Suspense 边界像一位前厅经理：子组件 render 时读到 pending 的 promise，直接把它 **throw** 给最近边界。边界打上 DidCapture 标记，先把 fallback 零食盘端出去，primary 主菜则以隐藏状态在后台挂起等待。

promise resolve 后，ping 回调唤醒根节点、调度一次 RetryLane 渲染；这次 thenable 已被标记 fulfilled，组件同步拿到值，主菜在 commit 阶段瞬间替换零食——等待被编码成控制流，组件只需写「数据就绪后」的 happy path。

两个细节别踩坑：rejected 的 promise 不归边界管，重放时 throw reason 交给错误边界；promise 必须来自缓存层或 transition，渲染中随手新建 promise 会因反复重放而无限挂起。

## 码叔划重点

1. 并发不是并行：单线程协作式调度，render 可中断、可丢弃、可重放，commit 始终同步不可打断。
2. useTransition 把更新标记为非紧急，被高优先级更新打断后丢弃旧渲染，基于新 state 重新渲染。
3. Suspense 中子组件把 promise 抛给边界，fallback 先行，resolve 后经 ping 以 RetryLane 重放渲染。

## 自测一题

**问：** 低优先级渲染做到一半被丢弃重做，之前的计算不就白费了？React 为什么还敢这么设计？

**答：** 被丢弃的只是 render 阶段在内存里构建的 fiber，它是纯计算、没有任何副作用，重来不会留下「半更新的界面」；真正不可撤销的 DOM 操作全部推迟到同步、不可打断的 commit 阶段。

相比让一次大渲染堵死主线程、用户输入连续掉帧，丢弃一段可重放的计算代价更小。并发优化的目标是用户感知的响应性而非总渲染量；再加上 transition 的 5 秒过期机制，让位永远不会演变成永久饿死。

一句话记住：**能放下的半成品，才敢随时重来；端上桌的那盘，必须一气呵成。**
