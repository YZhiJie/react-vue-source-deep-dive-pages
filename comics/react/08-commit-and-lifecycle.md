# 漫画 · 第 8 话 commit：不可打断的上菜时刻

> 对应正文：[第 8 章 commit 阶段与生命周期](../../docs/react/08-commit-and-lifecycle.md) ｜ 原画：[08-commit-and-lifecycle.svg](./08-commit-and-lifecycle.svg)

## 登场角色

- **阿R**：React 餐厅的服务员兼主厨，蓝色小机器人，胸口的「R」是它的骄傲。
- **阿V**：隔壁 Vue 绿餐厅的主厨兼老板娘，绿发少女，头顶别着 Vue 同款 V 字发饰。
- **码叔**：猫头鹰教授，每话结尾出来划重点。

## 剧情梗概

render 备完菜，终于到了上桌时刻。阿R 推开餐厅双开门——commit 阶段**同步、不可打断**：BeforeMutation 趁旧盘还在赶紧读快照，Mutation 动真格换餐具、改真实 DOM，Layout 量盘执行 layout effect；等浏览器绘制之后，Passive 才像餐后甜点一样慢悠悠端上。阿V 还在换景瞬间抓到一个关键动作：`root.current` 指针恰好在 Mutation 与 Layout 之间翻面。

这一话的道具都很直白：四块牌是时间轴，闸刀是指针切换点，旧碗筷与新碗筷是 effect 的 destroy 与 create。先记住一句话——commit 一旦开始，菜上到一半绝不能撤回厨房。

## 分格解读

### 第 1 格 · 上菜四步走

上菜分四步：BeforeMutation 读快照（`getSnapshotBeforeUpdate`、HostRoot 的 `clearContainer`），Mutation 执行 DOM 增删改与卸载，Layout 执行 `componentDidMount/Update` 与 `useLayoutEffect` 的 create——前三步同步连续、不可打断；Passive 由 `scheduleCallback` 另排普通优先级任务，等浏览器绘制后再跑 `useEffect`。

为什么 Passive 画成「餐后甜点」？因为 `commitRoot` 对它只调度、不执行：真正的 `flushPassiveEffects` 是两次独立的整树遍历，发生在页面 paint 之后。`useLayoutEffect` 里改 DOM 不闪屏，是因为 Layout 在绘制前已同步跑完；而 `useEffect` 读到的永远是「已经画出来」的世界。

这个顺序也解释了 `getSnapshotBeforeUpdate` 的独特位置：它在 BeforeMutation、DOM 还是旧的时候读状态，返回值存到实例上，稍后在 DOM 已更新的 `componentDidUpdate` 第三参里消费——滚动位置保持靠的就是这次「旧读新用」。

### 第 2 格 · 换景瞬间

牌上写的是 React 19.3 源码 ReactFiberWorkLoop.js 里最关键的一行：`root.current = finishedWork`，精确夹在 Mutation 结束与 Layout 开始之间。Mutation 中执行 `componentWillUnmount` 时旧树必须还是 current，卸载读到的是旧 props/state；Layout 中执行 `componentDidMount` 时新树必须已经是 current，此时 `setState`、ref 面对的都是新界面。

闸刀只能在这个点落下：早一拍，卸载逻辑读到的树已经换了，旧 ref、旧状态全失踪；晚一拍，CDM/CUD 里通过 ref 拿到的就是旧 DOM。源码注释把话说得很死：must come after the mutation phase, but before the layout phase。

顺带破除老资料：19.3 的入口函数就叫 `commitRoot`，`commitRootImpl` 已不存在；默认路径下 `flushMutationEffects` 与 `flushLayoutEffects` 同步直调，只有开启 View Transition 时才包进 `startViewTransition` 分段驱动。

### 第 3 格 · 先收桌再摆新碗筷

「先收桌再摆新碗筷」对应一条铁律：同一次提交里，**所有** fiber 的 effect destroy 全部跑完，才允许任何一个 create 开跑。Layout 侧靠「destroy 提前到 Mutation、create 留在 Layout」天然实现；Passive 侧靠两次独立整树遍历：先 `commitPassiveUnmountEffects`，再 `commitPassiveMountEffects`。

为什么不能「我的 destroy 紧邻我的 create」？源码注释给了答案：防止兄弟组件互相污染——一个组件的 destroy 绝不应该覆盖另一个组件 create 刚挂上的 ref。所以阿R 必须先把全桌旧碗收进托盘，才能统一摆上新碗。

遍历时 React 19 不再走 effect list 链表，而是直接递归 fiber 树，用 `subtreeFlags` 位掩码剪枝：子树标记里没有对应 Mask 就整棵跳过；`flags` 管自己、`subtreeFlags` 管后代，各阶段各用各的 Mask。另外记住两条次序：挂载时 create 子先父后，卸载时 destroy 父先子后。

## 码叔划重点

1. commit = 三个同步子阶段加一个延迟 Passive：读快照 → 改真实 DOM → 执行 layout effect → 浏览器绘制后再跑 passive effect。
2. `root.current` 在 Mutation 与 Layout 之间切换：卸载读旧树、挂载读新树，时机差一拍都不行。
3. React 19 用 `subtreeFlags` 位掩码递归剪枝，不再构建 effect list；同一次提交所有 destroy 先于所有 create。

一句话记住这顿饭：render 是可以重来的备菜，commit 是不可撤销的上菜——「能撤销的计算」和「不能撤销的副作用」被 React 用阶段边界严格隔开。

## 自测一题

**问：** `useEffect` 的回调为什么可能在浏览器绘制之后才执行？如果在它里面同步测量 DOM 位置并校正滚动条，用户会看到什么？

**答：** `useEffect` 属于 Passive 阶段，`commitRoot` 只用 `scheduleCallback` 排一个普通优先级任务，要等 Layout 结束、浏览器 paint 完之后才 flush，且先整树 destroy 再整树 create。在它里面校正滚动条，用户可能先看到一帧跳动再被拉回；绘制前必须完成的测量与校正应改用 `useLayoutEffect`，它在 Layout 阶段同步执行、阻塞绘制。

还要注意销毁次序：不要假设「我的 cleanup 紧接着我的下一个 create」。更新时 React 先跑完所有组件的 destroy 再跑所有 create；类组件卸载更是先解绑 ref、再调 `componentWillUnmount`，在 CWU 里通过 props 回调往外传 ref 只会拿到 null。
