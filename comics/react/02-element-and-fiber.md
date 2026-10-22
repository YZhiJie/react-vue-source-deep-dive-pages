# 漫画 · 第 2 话 Element 与 Fiber：剧本与演出档案

> 对应正文：[第 2 章 ReactElement 与 Fiber 架构](../../docs/react/02-element-and-fiber.md) ｜ 原画：[02-element-and-fiber.svg](./02-element-and-fiber.svg)

## 登场角色

- **阿R**：React 剧团的台柱机器人，本话兼职编剧与舞台监督，一手剧本一手档案。
- **阿V**：隔壁 Vue 绿餐厅的主厨兼老板娘，今天溜进剧场看戏，专问刁钻问题。
- **码叔**：猫头鹰教授，每话结尾出来划重点。
- **道具**：剧本卷轴、Fiber 档案卡、换景闸刀，本话的三件关键道具。

## 剧情梗概

阿V 来阿R 的剧团探班，发现剧团里跑着两套东西：演员手里的**剧本**写完就不许再改，而舞台监督阿R 手里的**演出档案**却写满了批注。

看戏看到一半，阿R 扳下舞台边的换景闸刀——原来台上台下一直备着两套布景，这正是演出随时能暂停、能重排的底气。

## 分格解读

### 第 1 格 · 剧本 ReactElement

JSX 编译后得到的 ReactElement 就是一份「剧本」：一个不可变、无状态的普通对象，只描述「页面应该长这样」。

它身上带着 `$$typeof` 身份戳和 `type`、`key`、`props` 等字段，dev 下还会被整体 freeze——写完即定稿。

想改戏？只能参照旧剧本写一本新的（创建新元素），绝不允许在旧本上涂涂改改。元素是一次性快照，用完即弃。

### 第 2 格 · 演出档案 Fiber

Fiber 才是 React 内部真正干活的**工作单元**：一份可变的、长期维护的「演出档案」。

档案之间靠 `child`、`sibling`、`return` 三根指针串成链表树，外加新旧 props、状态、flags 标记和 alternate 替身。

每次 render，React 拿新剧本在档案上「对账」（reconcile）：元素用完即弃，Fiber 却被复用和修改——剧本常换，档案长存。

### 第 3 格 · 双缓存两棵树

舞台上永远有两套布景：**current 树**正在上演（对应当前屏幕），**workInProgress 树**在后台排练（内存中构建）。

同一角色在两树各有一份档案，互以 `alternate` 指向对方；render 的全部改动都先打在 WIP 树上。

排练完成后扳下闸刀——`FiberRoot.current` 指针一次切换，WIP 树转正上演，旧树降级为下一轮可复用的替身池。这就是 render 可中断的底气。

## 码叔划重点

1. Element 是不可变剧本：只描述演什么，用完即弃。
2. Fiber 是可变演出档案：child/sibling/return 三指针串起的工作单元。
3. 双缓存：后台排练 workInProgress，换景时 current 一次切换。

## 自测一题

**问：** 为什么 React 要同时维护 current 和 workInProgress 两棵 Fiber 树，直接在一棵树上改不行吗？

**答：** 不行。如果直接在用户正看着的 current 树上改，render 一旦被打断，屏幕就会半新半旧。

有了双缓存，React 可以安心在内存里的 WIP 树上构建，随时暂停、重来甚至整棵丢弃；等排练完毕，commit 阶段把 `FiberRoot.current` 一指，新树瞬间转正，用户永远只看到完整的画面。
