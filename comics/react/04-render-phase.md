# 漫画 · 第 4 话 render 阶段：可中断的深度优先遍历

> 对应正文：[第 4 章 render 阶段——可中断的深度优先遍历](../../docs/react/04-render-phase.md) ｜ 原画：[04-render-phase.svg](./04-render-phase.svg)

## 登场角色

- **阿R**：React 餐厅的主厨，蓝色小机器人，本话戴上厨师帽在备菜间展示「随时能放下」的备菜功。
- **码叔**：猫头鹰教授，每话结尾出来划重点。

## 剧情梗概

调度台叫号之后，阿R 带我们走进 render 备菜间。这里的规矩只有一条：**菜没端给客人之前，备菜随时可以暂停、可以整盘倒掉重来**；可一旦进入 commit 上菜，就必须一口气做完，绝不能半途而废。

备菜本身是一次沿 fiber 树的深度优先遍历：阿R 顺着树一路「递」下去找活干（beginWork），触底后再一路「归」上来收副作用（completeWork），途中还会横着挪到兄弟节点继续递。

遍历结束，托盘上只多了一棵 finishedWork 草稿树和一张副作用清单——厨房里算得热火朝天，餐厅里的真 DOM 一行都没变。

## 分格解读

### 第 1 格 · 备菜能停，上菜不能停

阿R 一边切菜一边强调两阶段的分工。

render 是**备菜**：在内存里调用组件、做 Diff、涂改 workInProgress 草稿树，产物不挂到页面上，所以时间片用完随时能放下菜刀，并发循环里每干完一个单元就抬头问一句 shouldYield。

commit 是**上菜**：真正执行 DOM 增删改、ref 与 effect，同步且不可打断——宫保鸡丁不能上半盘再撤回去加花生。可中断、可丢弃的纯计算，与不可撤销的副作用，就这样被严格分成两段。

### 第 2 格 · 沿 fiber 树爬上爬下

备菜的走位是一条 U 型轨迹，由模块级指针 workInProgress 驱动，它永远指向当前正在处理的节点：

- **beginWork（递）**：处理当前节点后返回它的长子，指针一路向下找活干；返回 null 说明触到叶子，开始收尾。beginWork 内部先按 tag 分派（函数组件、HostComponent 各有各的干法），还会做两层 bailout，能跳过的子树连孩子都不克隆。
- **completeWork（归）**：叶子先收尾，然后有兄弟就横移过去 begin 兄弟，没兄弟才回到父亲 complete 父亲。每个节点的 flags 与子树的 subtreeFlags 随归程一层层向上冒泡。

递是先序、归是后序，合起来正是一棵完整的深度优先遍历树；指针一路回到根，render 才算 RootCompleted。

### 第 3 格 · finishedWork 出餐托盘

遍历走完，托盘上的成品不是一桌真菜，而是 finishedWork 草稿树和一张**副作用清单**：

- Update：props 引用变了，commit 时要更新属性；
- Placement：新增的节点，commit 时要插入文档；
- subtreeFlags：子树里都有哪几类副作用，供 commit 时拿着 masks 剪枝，没事的子树直接不进。

清单上的红章盖得清楚：**一行 DOM 没动**。挂载时 completeWork 里 createInstance 造出来的只是游离 DOM，父子关系先在内存里拼好，真正插入文档的 Placement 动作要留给 commit 阶段。

## 码叔划重点

1. render 是可中断的深度优先遍历：beginWork 向下分派、completeWork 向上归集，兄弟横移、无兄回父。
2. completeWork 计算 props 差异并打上副作用标记（flags），再用 bubbleProperties 把 subtreeFlags 冒泡给祖先，产出 finishedWork 树。
3. 全程纯计算、不碰真 DOM：中断了从断点恢复，被高优先级更新插队就丢弃草稿、从根重来。

## 自测一题

**问：** render 被打断后为什么能从断点继续，而被高优先级更新插队时却要整棵树丢弃重来？

**答：** 进度全部记在模块级变量 workInProgress 上。时间片用完只是退出工作循环，根和 lanes 都没变，下次进入不重置现场，指针还指着没做完的节点，自然从断点继续。

而高优先级更新会让本次渲染的 lanes 发生变化，prepareFreshStack 会重新建草稿根，旧 workInProgress 树连同里面已经造好的游离 DOM 整体作废、交给 GC，从根重新遍历。正因为 render 全程不碰真 DOM，「丢弃重来」对用户才完全不可见——这也是 render 函数体里严禁写副作用的原因。
