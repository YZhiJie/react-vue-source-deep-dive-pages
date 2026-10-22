# 漫画 · 第 3 话 Scheduler 与 Lane：急诊分诊台

> 对应正文：[第 3 章 Scheduler 调度器与 Lane 优先级模型](../../docs/react/03-scheduler-and-lanes.md) ｜ 原画：[03-scheduler-and-lanes.svg](./03-scheduler-and-lanes.svg)

## 登场角色

- **阿R**：React 餐厅出身的分诊护士，本话戴上红十字护士帽，守在急诊分诊台后按轻重缓急叫号。
- **阿V**：从隔壁 Vue 绿餐厅跑来挂号的急病人，满头大汗，举着「我先！」的牌子想插队。
- **码叔**：猫头鹰教授，每话结尾出来划重点。

## 剧情梗概

阿V 捂着肚子冲进急诊分诊台大喊「我先」，阿R 却不按先来后到发号——在 React 医院里，病人挂号按**轻重缓急**分诊：用户输入、点击是「急诊」，startTransition 包起来的转场动画是「慢诊」。

分诊台之后，阿R 带我们参观地下的 31 条车道（Lane 模型），最后掏出怀表揭秘时间切片：每干 5ms 就把主线程还给浏览器，而报时的「闹钟」正是 MessageChannel 宏任务。

## 分格解读

### 第 1 格 · 分诊台不按先来后到

医院急诊的铁律是**分诊**：心梗先于包扎，包扎先于体检。React 同理——setState 不等于立刻渲染，更新先被贴上一张「分诊标签」（Lane）：输入、点击落在 SyncLane 一带，随时插队；startTransition 包起来的大列表渲染落在 TransitionLane，甘愿排队。

这座分诊台由三套机制组成：通用任务队列 **Scheduler**（叫号系统，与 React 无关）、reconciler 内部的 **Lane**（分诊标签），以及把两者接起来的 **ReactFiberRootScheduler**（分诊台本身）。含 SyncLane 的更新在微任务末尾同步刷掉，其余车道翻译成 Scheduler 优先级后排队切片。

慢诊也不会被无限饿死：`markStarvedLanesAsExpired` 给每条 lane 登记过期时刻，等太久就并入 expiredLanes「重新提级」，下次渲染放弃切片、一口气同步跑完。

### 第 2 格 · Lane 车道模型

Lane 不是比数字大小，而是 **31 个二进制位**组成的车道集合：位越靠右（数值越小）优先级越高。SyncLane、InputContinuousLane、DefaultLane 在右侧，14 条 TransitionLane 居中，Retry、Idle、Offscreen、Deferred 在最左。

一个整数就能同时挂多条车道：合并是按位或，判交集是按位与，取最高优先级只要 `lanes & -lanes`（补码取最右一个 1）——全是 O(1) 位运算。Fiber 上的 lanes 与 childLanes 还能让 render 阶段整棵子树剪枝：位上没活儿的子树，连进都不用进去。

每次渲染前 `getNextLanes` 按「非 Idle → 新鲜 → ping 唤醒 → 预热」选批，同级或更低优先级的新车道不许打断在跑的渲染；被 entangle 纠缠的车道则必须同生共死、同批渲染。

每种交互车道旁边还配一条 Hydration 车道，专供 SSR 注水；Idle 工作必须等所有非 Idle 车道清空后才轮得到，是真正的「慢诊末班」。

### 第 3 格 · 时间切片 5ms 一让

阿R 的怀表就是 `shouldYieldToHost`：每轮工作预算 **5ms**，预算用完、或浏览器有一帧等着绘制，就立刻让出主线程；没干完的活返回一个续算函数，由 MessageChannel 的 postMessage 宏任务排到下轮接着跑。

为什么闹钟选 MessageChannel 而不是 setTimeout？嵌套超过 5 层的定时器会被浏览器钳制到至少 4ms，切片一轮预算才 5ms，光唤醒就耗掉大半；requestIdleCallback 在后台标签页又会被推迟甚至冻结。MessageChannel 零延迟、节拍稳定，于是点击输入永远能插队，渲染再久也不卡交互。

注意「过期」不是被取消，而是被允许霸占主线程：workLoop 里未过期任务受 5ms 预算约束，过期任务（didTimeout）则不再让出、同步跑完——这是防饥饿的最后保险。

## 码叔划重点

1. Scheduler 用最小堆管理任务：堆键是过期时间，到期先跑；同一优先级再比入队 id，先来先做。
2. Lane 是 31 位二进制优先级模型：位越小越急，支持按位合并与批量剔除，交并子集全是位运算。
3. 时间切片借 MessageChannel 宏任务让出主线程：每 5ms 一让，点击输入永远比渲染先响应。

## 自测一题

**问：** Scheduler 为什么不选 setTimeout(fn, 0) 或 requestIdleCallback 来驱动下一轮时间切片？

**答：** setTimeout 嵌套超过 5 层后会被浏览器钳制到至少 4ms 才触发，而切片预算总共才 5ms，光等唤醒就耗掉大半，节拍完全被打乱；requestIdleCallback 在后台标签页会被推迟甚至冻结，触发时机不稳定，给不了「每 5ms 必到」的保证。

MessageChannel 的 postMessage 是零延迟宏任务，不受钳制、不看标签页脸色，正好当切片的节拍器（Node.js 环境则优先用 setImmediate 兜底）。
