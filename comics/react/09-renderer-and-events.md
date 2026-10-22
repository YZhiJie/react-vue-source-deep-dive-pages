# 漫画 · 第 9 话 渲染器与事件：三层楼与总接待台

> 对应正文：[第 9 章 渲染器与事件系统](../../docs/react/09-renderer-and-events.md) ｜ 原画：[09-renderer-and-events.svg](./09-renderer-and-events.svg)

## 登场角色

- **阿R**：React 餐厅的服务员兼主厨，蓝色小机器人，胸口的「R」是它的骄傲。
- **阿V**：隔壁 Vue 绿餐厅的主厨兼老板娘，绿发少女，头顶别着 Vue 同款 V 字发饰。
- **码叔**：猫头鹰教授，每话结尾出来划重点。

## 剧情梗概

阿V 来参观阿R 新盖的三层小楼：`react` 是前台 API 薄壳，`reconciler` 是宿主无关的调度内核，`react-dom` 是懂 Web 的渲染器，两层之间靠一份叫 host config 的契约连接。楼下大堂还设了「总接待台」——事件不再层层挂监听，而是在 root 容器统一委托；原生事件敲门后，沿 fiber 的 return 链收集 listener，再模拟出捕获与冒泡两阶段派发。

## 分格解读

### 第 1 格 · 三层楼

React 分三层：`react` 包只有 `useState`、`createElement` 这些 API 表面，运行时由内核注入实现；`react-reconciler` 装着 fiber、lane、render/commit、hooks，却**完全不认识 div**；`react-dom`（19.3 中实际代码在 `react-dom-bindings`）负责 `createRoot`、DOM 操作与合成事件。

内核与渲染器之间是一份契约：reconciler 在固定时机调用 host config 约定方法（`createInstance`、`appendChild`、`commitUpdate`……），react-dom 给出 DOM 实现。内核因此宿主无关——React Native、Ink、Three.js 只要实现同一套接口，就能复用整座楼。

这份契约还决定了 DOM 出现的时机：completeWork 用 `createInstance` 和 `appendInitialChild` 在内存里把整棵子树拼好，直到 commit 才一次性 `appendChild` 进文档；属性 diff 也不在 render，而是 commit 时由 `commitUpdate → updateProperties` 现算。

### 第 2 格 · 大堂总接待台

`createRoot(container)` 内部做三件事：建 `FiberRoot`（模式恒为 `ConcurrentRoot=1`）、在容器 DOM 上登记根标记、调 `listenToAllSupportedEvents` 给每种事件挂监听。click、keydown、input……所有事件线都汇聚到 root 容器这一个接待台。

每种事件默认挂**两个**委托监听：一个 bubble、一个 capture；且容器上有 listeningMarker 去重，多根树各挂各的、互不干扰。接待台还会按事件类型选包装器：click 这类离散事件在派发期间把更新优先级提为 Discrete，其中的 setState 才总是比 transition 更新先渲染。

委托挂容器而不是旧版的 document：一个页面可以有多个 React 根，挂容器天然隔离；第三方代码拦住冒泡到 document 也影响不到 React；容器外的 DOM 本来就不归本根管。图里被红叉划掉的 document 牌，就是 React 17 起改址、19 依旧的旧门牌（`selectionchange` 等极少数不冒泡事件仍挂 document）。

### 第 3 格 · 冒泡寻人

原生事件只认识 DOM。接待台先用 `getClosestInstanceFromNode` 从目标节点反查 fiber（挂载时 DOM 节点上存了指向 fiber 的内部 key），再让 7 个插件依次 `extractEvents`：SimpleEventPlugin 从目标 fiber 起，每经过一个 HostComponent 就取它 props 上的 `onClick` / `onClickCapture`，然后 `instance = instance.return`，一路收集到根。

同一份 listener 数组走两遍：capture 趟**倒序**遍历（根 → 目标），bubble 趟**正序**遍历（目标 → 根），每次派发前设置 `event.currentTarget`，handler 一返回立刻置空；`stopPropagation` 则让循环提前退出。所以图上的上下两条箭头，其实是内核沿 return 链手工模拟的两阶段，不是浏览器在层层 DOM 上原样冒泡。

其余 6 个插件是浏览器行为的垫片：ChangeEventPlugin 靠 `_valueTracker` 判断受控 input 的值是否真的变化，合成出语义接近原生 `input` 的 `onChange`；EnterLeaveEventPlugin 用 `mouseout`/`mouseover` 算出不冒泡的 `onMouseEnter`/`onMouseLeave`。

## 码叔划重点

1. 内核宿主无关：react-dom 用 host config 一组方法反向提供 DOM 能力，换平台只需实现同一套接口。
2. React 19 事件委托在 root 容器而非 document，多根树互不干扰（`selectionchange` 等极少数例外仍挂 document）。
3. 合成事件从目标 fiber 沿 return 链收集 listener，同一数组倒序/正序各走一遍，模拟捕获与冒泡两阶段。

容易看漏的一点是：监听只挂容器、listener 却存在每个 HostComponent 的 props 上，`commitUpdate` 更新属性时会顺手 `updateFiberProps`，保证事件系统拿到的永远是最新一版 handler——这也是委托模式不用重新绑监听的原因。

## 自测一题

**问：** 既然监听只挂在 root 容器上，React 怎么知道该调哪个组件的 onClick？为什么 `e.persist()` 可以不调，而异步读 `e.currentTarget` 还是 null？

**答：** 派发时先由 DOM 目标反查 fiber，再沿 fiber.return 一路收集每个 HostComponent props 上的 handler，所以不必逐节点挂监听也能找齐 onClick 与 onClickCapture，随后按 capture 倒序、bubble 正序派发。19 里事件池早在 React 17 就移除了，`persist()` 只剩兼容空壳；但 `currentTarget` 是派发态字段，每个 listener 执行前设置、执行后立刻置空，异步读取必然是 null，需要先 `const el = e.currentTarget` 存到局部变量。

顺带一提，click、keydown 这类离散事件在包装器里把更新优先级提为 DiscreteEventPriority，其中的 setState 走高优 lane、总是先于 transition 提交；mousemove、drag 这类连续事件则允许合并与让步。

`scroll`/`scrollend` 是「沿 return 链收集」的例外：浏览器里它们不冒泡，React 收集到目标节点后用 accumulateTargetOnly 直接截断，listener 只取目标节点自己的，不会继续向上寻人。
