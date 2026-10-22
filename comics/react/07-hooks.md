# 漫画 · 第 7 话 Hooks：只认顺序的钩子链

> 对应正文：[第 7 章 Hooks 系统原理](../../docs/react/07-hooks.md) ｜ 原画：[07-hooks.svg](./07-hooks.svg)

## 登场角色

- **阿R**：React 餐厅的服务员兼主厨，蓝色小机器人，胸口的「R」是它的骄傲。
- **阿V**：隔壁 Vue 绿餐厅的主厨兼老板娘，绿发少女，头顶别着 Vue 同款 V 字发饰。
- **码叔**：猫头鹰教授，每话结尾出来划重点。

## 剧情梗概

阿V 为了少写一行代码，把 `useState` 塞进了 `if`。第二次渲染时，阿R 手里的钩子链少了一环，第二个环怎么也找不到对应的人，当场抓狂。稳住情绪后，阿R 带她参观自己的衣帽间：`react` 包只是薄壳，真正干活的是按渲染阶段换上的四套 Dispatcher 工服。最后两人站在 effect 环形链前看一盏开关灯——这就是 `HasEffect` 位：依赖没变，这盏灯这次不亮，effect 入了环也不执行。

## 分格解读

### 第 1 格 · 错位事故

Class 组件的 `this.state` 是对象，靠 **key** 存取，顺序无所谓；函数组件每次渲染只是一次普通函数调用，执行完栈帧就销毁。React 的办法极其朴素：在 `fiber.memoizedState` 上维护一条**按调用顺序排列的 hook 单向链表**——第一次 `useState` 对应第 1 个节点，第二次对应第 2 个，只认位置不认名字。

阿V 把 `useState` 写进 `if`，下一次渲染这个 Hook 就时有时无。更新时每个 wip hook 都要从 `current.alternate` 的对应节点浅克隆，位置一旦对不上、走到链表尽头找不到节点，就会抛出 `Rendered more hooks than during the previous render.`。所以 Rules of Hooks **不是语法规定，而是数据结构约束**。

画里阿R 的链子上第三个环滑出了队列（红色错位环 + 灰色虚线虚位），正是「第二个环找不到人了」。记住 hook 链**不成环**，尾部 `next` 为 null，和格 3 的 effect 环恰好相反。要绕开规则，只能把条件放进 Hook 内部（比如在 effect 回调里判断），不能把 Hook 放进条件。

### 第 2 格 · 四件工服

`import { useState } from 'react'` 拿到的并不是真实现：`react` 包里每个 Hook 只做「取 dispatcher 再转发」两件事，真正的实现由 `react-reconciler` 在渲染期间注入模块级槽位 `ReactSharedInternals.H`。这层薄壳让同一个 `react` 包可以搭配 react-dom、React Native、Test Renderer。

衣架上四件工服就是四套 Dispatcher：`HooksDispatcherOnMount` 首次挂载（`useState` 指向 `mountState`）、`HooksDispatcherOnUpdate` 正常更新（`updateState`）、`HooksDispatcherOnRerender` 负责 render 阶段重渲染与 Suspense 重放（`rerenderState`）、`ContextOnlyDispatcher` 在不在任何组件渲染中时上场（所有 Hook 都指向 `throwInvalidHookError`，只有 `useContext` 与 `use` 相关入口除外）。

换装发生在 `renderWithHooks`：`current` 为空就发 Mount 工服，否则发 Update；组件函数一执行完，`H` 立刻被换回 `ContextOnlyDispatcher`。所以在事件回调、定时器里直接调 `useState()` 看到的经典 `Invalid hook call`，不是 Hook 坏了，而是阿R 已经下班换回了便服。

### 第 3 格 · 带位开关

`useEffect` 的记录挂在另一条结构上：`fiber.updateQueue.lastEffect` 指向的 **effect 环形链表**。第一个 effect 自环，之后按标准环插，永远成环，方便从任意位置插入与遍历。每条记录带着 `tag`、`create`、`deps`、`inst`、`next` 五个字段。

`tag` 是位标记而不是枚举：`HasEffect = 1` 是「本次真的执行」的开关位，`Insertion = 2` / `Layout = 4` / `Passive = 8` 是类别位。`useEffect` 首次挂载无条件带 `HasEffect`（tag 即 `1001`）；更新时由 `areHookInputsEqual` 逐个用 `Object.is` 比依赖——依赖没变，effect 记录**照样推入环**（要持有新的 `create` 闭包和共享的 `inst`），但 tag 只剩类别位、没有 `HasEffect`，fiber 也不打 flags。

于是 commit 阶段遍历环时只执行带 `HasEffect` 位的记录，开关灯灭的那一环直接跳过。这解释了两个日常现象：依赖数组不变时 effect 不执行；**不传**依赖数组时每次都带 `HasEffect`、每次都执行，而 `[]` 才是只在挂载时执行一次。

## 码叔划重点

1. 状态挂在 `fiber.memoizedState` 的 hook 链表上，只认调用顺序，不能写在条件或循环里。
2. `react` 包只是薄壳，四套 Dispatcher 按渲染阶段注入真实实现，渲染一结束就换回 `ContextOnly`。
3. `useEffect` 靠 tag 类别位与 `HasEffect` 开关位标记，决定本次 commit 是否真正执行。

顺带记牢两条链的形状：hook 链是单向链、尾部为 `null`；effect 链必须成环，入口是 `updateQueue.lastEffect`。还有个特例 `useContext`——它不进 hook 链，依赖单独挂在 `fiber.dependencies` 上。

## 自测一题

**问：** 把 `useState` 写进 `if` 为什么会崩？在点击事件回调里直接调 `useState()` 报的 `Invalid hook call` 又是谁抛的？

**答：** 更新渲染要按调用位置从 current 树逐个克隆 hook 节点，`if` 让本次调用次数发生变化，第 N 个位置在旧链上找不到对应节点，就抛 `Rendered more hooks than during the previous render.`。组件函数执行完后 `ReactSharedInternals.H` 被重置为 `ContextOnlyDispatcher`，它的 `useState` 指向 `throwInvalidHookError`，所以事件回调里调用必然报错。一个是链表错位，一个是没有 Dispatcher，两道防线别记混。

追加一问：`useEffect(fn, [])` 首次挂载和第二次渲染的 tag 分别是什么？首次是 `HasEffect | Passive`（`1001`）所以执行；第二次依赖没变，只剩 `Passive`（`1000`），开关灯灭、本次跳过。

再补一个冷知识：React 19 里只有 `use` 可以写进条件或循环——它不向 hook 链表追加节点，接 Promise 走 thenable 抛接，接 Context 则直接 `readContext`。其余 Hook 谁都没有这个豁免权。
