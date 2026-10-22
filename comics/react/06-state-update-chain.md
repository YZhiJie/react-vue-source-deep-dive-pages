# 漫画 · 第 6 话 状态更新链路：setN 的奇幻漂流

> 对应正文：[第 6 章 状态更新完整链路](../../docs/react/06-state-update-chain.md) ｜ 原画：[06-state-update-chain.svg](./06-state-update-chain.svg)

## 登场角色

- **阿R**：React 餐厅的服务员兼主厨，蓝色小机器人，胸口的「R」是它的骄傲。
- **阿V**：隔壁 Vue 绿餐厅的主厨兼老板娘，绿发少女，头顶别着 Vue 同款 V 字发饰。
- **码叔**：猫头鹰教授，每话结尾出来划重点。

## 剧情梗概

阿V 在按钮上连点两下：`setN(1)` 紧接着 `setN(x ⇒ x＋1)`。她疑惑：为什么只渲染一次？为什么马上读 `n` 还是旧值？阿R 带她看订单的「奇幻漂流」——每次 setState 只是造一个 Update 对象挂上环形链表，按 lane 排队，render 时才统一消费；被跳过的慢单也不会丢，下次基于 baseState 重放，最终态分毫不差。

## 分格解读

### 第 1 格 · 两单并一灶

同一事件里连调两次 `setN`，React 并不立刻算新 state，而是各造一个带 lane、payload、next 的 **Update** 对象，挂上 `shared.pending` **环形链表**——尾指针让头尾都能 O(1) 拿到。两次更新同队按序应用（函数式 updater 会基于上一条的结果再算），共享同一个调度边界，所以只渲染一次。这就是自动批处理：批处理由调度模型天然保证，19 里 `batchedUpdates` 已是 noop。

阿R 把两张订单钉在同一环形队列转盘上，本质对应正文里 enqueue 后的 `shared.pending` 指针：pending 永远指向最新更新，pending.next 就能拿到最老的那条。处理时断环拼 base 链，再进入 do/while 逐条消费。第一次渲染时，两张订单同一 lane 同时出锅，所以阿V 只会看到一次上屏，而不是两次。

更细一点：19 的正常路径下更新并不直接挂进 fiber 的队列，而是先以 `(fiber, queue, update, lane)` 四元组暂存到模块级的 `concurrentQueues` 数组里，等 render 前后安全边界再统一 drain 挂环。这个「先暂存、后统一」的设计让事件回调里的多次 setState 天然合批。

### 第 2 格 · 旧值之谜

`setN(1)` 之后立刻 `console.log(n)` 读到的仍是 0——setState 不是「立即改 state」，只是入队；新 state 要等 render 阶段 `processUpdateQueue` / `updateReducerImpl` 消费队列时才算出来。事件回调里读到的是旧闭包。函数组件还有 **eagerState** 优化：fiber 没有挂起更新时先拿 `lastRenderedReducer` 试算一遍，结果 `Object.is` 相同就直接 bailout，连调度都省了。

这正是正文反复提醒的一点：以为 setState 是「立即改 state」是初学者最常踩的坑。订单只是存进队列，render 到后厨窗口消费时才算数，上屏更是 commit 的事。阿V 手里的小票还写着 `n = 0`，因为闭包捕获的是渲染当时的快照，要等下一轮 render 才会刷新。

eagerState 有个重要前提：只有 fiber 当前没有任何待处理更新（`fiber.lanes` 为空）时试算才安全。如果已有更新在排队，最终 state 必须以队列重放为准，提前试算可能算错，所以这种情况下会跳过优化直接入队。

### 第 3 格 · 插队与重放

队列按 **lane** 分诊：急单（高优先级）先做，慢单被跳过——但跳过不是删除。被跳过的更新会克隆进新的 base 队列，且其后的高优更新也要连同它一起基于更早的 **baseState 重放**（正文 6.4.1 的 A1-B2-C1-D2 模型）。这样无论中断重放多少次，中间态可随资源变化，最终 state 一定等于「按入队顺序全部应用」的结果——这就是 transition 可被打断却不出错的公平性设计。

这也是 `concurrentQueues` 存在的意义：render 进行中如果事件把更新直接插进链表，正在迭代的顺序会全部乱套。所以先暂存四元组，在 prepareFreshStack 后或 render 完成后的安全边界统一 drain 挂环，再沿 return 上溯标 childLanes。最后由 `scheduleUpdateOnFiber` 调 `markRootUpdated` 把 lane 并进 `root.pendingLanes`，交给 `ensureRootIsScheduled` 去重并安排下一次任务。

源码里 `processUpdateQueue` 的核心判断是 `isSubsetOfLanes(renderLanes, updateLane)`：当前 render 的 lanes 集合包含这条更新的 lane 才消费，否则克隆进新 base 队列、锁定 newBaseState。它后面哪怕再来一条 lane 很高的更新，只要前面有慢单被跳过，这条高优单也会被克隆成 NoLane 进 base 队列，下次必处理——这正是「公平性」的实现。

## 码叔划重点

1. Update 挂在环形链表上按 lane 消费：被跳过的低优先级更新留到下次重放，最终态确定。
2. eagerState 提前算第一个更新：值没变就 bailout，省掉一整轮 render。
3. 完整链路：setState 入队 → markRootUpdated → ensureRootIsScheduled → render → commit。

这套链路里最容易被忽略的一环是 `ensureRootIsScheduled`：它不只是「安排一次任务」，还负责任务去重——同一个 root 上瞬间产生多少更新，通常只保留一个调度任务。所以阿V 连按两下，RootScheduler 也只会安排一次 render，这就是自动批处理的底层支撑。

## 自测一题

**问：** 为什么 `setN(1)` 之后马上 `console.log(n)` 还是旧值？连续 `setN(1)` 和 `setN(x ⇒ x＋1)` 最终 n 是几？

**答：** setState 只是创建 Update 入队，state 在 render 阶段才重算，事件回调里读到的是旧闭包，所以 log 仍是 0。两条更新进同一条环形队列按序应用：先置为 1，再用函数式 updater 基于 1 加一，一次渲染后 n 最终为 2。如果先跳过高优更新再重放，低优更新也会基于锁定后的 baseState 重放，保证结果一致。

顺带一提：如果想强迫 React 立刻同步刷新，可以用 `flushSync`——它会强制 Discrete 优先级、立刻 drain 并同步跑完 render + commit。但这是有代价的逃生舱：打破自动批处理节奏、可能额外触发一次同步渲染，正文 6.10 明确建议只在第三方 DOM 库等必须同步读结果时使用。
