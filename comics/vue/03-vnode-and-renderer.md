# 漫画 · 第 13 话 VNode 与渲染器：拼一桌模型

> 对应正文：[第 3 章 VNode 与渲染器](../../docs/vue/03-vnode-and-renderer.md) ｜ 原画：[03-vnode-and-renderer.svg](./03-vnode-and-renderer.svg)

## 登场角色

- **阿V**：Vue 绿餐厅的主厨，这一话不上灶，改在后厨拼「VNode 模型桌」，手特别稳。
- **阿R**：React 餐厅的机器人服务员，又来串门，盯着 Vue 的「跳过」眼罩满脸问号。
- **码叔**：猫头鹰教授，每话结尾出来划重点。

## 剧情梗概

更新一道菜，为什么只有 `span` 里的数字变了、按钮纹丝不动？阿V 把秘诀拆成三步表演：模板卡片经 **render 函数**拼成一桌 **VNode 模型**；更新时渲染器拿新旧两桌做 **patch**，按 **ShapeFlags** 分派、只换掉不一样的零件；而 **Block** 手里攥着 `dynamicChildren` 平级名单，静态节点戴着眼罩直接跳过，看都不看一眼。

一句话路线：模板（编译期贴标签）→ render（产出 VNode 树与动态名单）→ patch（读标签靶向更新 DOM）。本章盯紧后两环，全部对应 Vue 3.5.43 源码。

## 分格解读

### 第 1 格 · 菜单变模型：render 跑出一桌 VNode

模板卡片（`div 卡片 / span 数字 / button 加一`）丢进 render 函数，跑一遍就拼出一棵圆球树——这就是 VNode。它不是真实 DOM，只是一个普通 JS 对象：`type` 说我是谁、`props` 装属性事件、`children` 挂孩子。

`h()` 是手写渲染用的创建工厂，编译器则直接调 `createVNode()`，建厂时顺手完成 class、style、key、ref 的归一化，并按类型烙上 `shapeFlag`。所以 ReactElement 与 VNode 本质同构，差别在 Vue 的 VNode 出厂就带编译情报。

- `type`：标签字符串、组件对象，或 Text、Fragment 等 Symbol 虚拟类型。
- `shapeFlag`：前 3 位是节点类型、中 3 位是孩子形态，一次按位与就完成判定。
- 注意：脱离 app 手动 `createVNode('div')` 只是个对象，挂载前 `el` 为 null，不会产生任何 DOM。

### 第 2 格 · patch 对比两桌：按 ShapeFlags 分派

更新来了，render 再跑一遍产出新桌 VNode（n2），渲染器的 `patch(n1, n2, container)` 把新旧两桌并排摆开：先看 `isSameVNodeType`（type 与 key 全等），再读 `shapeFlag` 位标志分派——元素走 `processElement`、文本走 `processText`、组件走 `processComponent`。

分派之后，旧桌没有就挂载（`mountElement`：先建游离元素、再挂孩子、再设 props、最后 insert），旧桌存在就更新（`patchElement` 复用同一个 DOM）。阿V 托盘中那个零件，正是两桌之间唯一对不上的圆球：其余零件原样摆回，只换它一个。

- 渲染器由 `createRenderer(options)` 工厂创建：runtime-core 不认识 document，节点操作全靠 runtime-dom 注入。
- 身份判定只认 type 加 key：`isSameVNodeType` 不成立就先卸载旧树，再按新节点挂载。
- 先在游离节点上组装好再一次性 insert，是为了减少多次布局与重排。

### 第 3 格 · 只有 span 变了：Block 只看 dynamicChildren

一棵 1000 个节点的模板可能只有 3 个会变。编译期给会变的节点贴彩色 **PatchFlag** 标签（TEXT 等于 1、CLASS 等于 2、STYLE 等于 4、PROPS 等于 8），render 执行时 `openBlock` 开张，把这些动态节点顺手收集进 `dynamicChildren` 平级数组。

更新时 Block 不做全树递归，只按下标遍历这个数组：动态 `span` 被放大镜逐格检查，静态的 `h1` 和 `button` 戴着「跳过」眼罩连读都不读。props 也按 flag 靶向更新，`PROPS` 配合 `dynamicProps` 连静态属性的 key 都不碰。

- 收集条件：`patchFlag` 大于 0，或者节点本身是组件（NEED_HYDRATION 除外）。
- 编译器保证 Block 内动态节点数量与位置稳定，所以 `patchBlockChildren` 连 key 都不比，直接下标一一对应。
- `v-if`、`v-for` 会另开新 Block，形成嵌套的 block 树。

## 码叔划重点

1. VNode 是描述界面的普通 JS 对象，`h()` 是创建工厂；ReactElement 与它同构，Vue 额外自带 shapeFlag、patchFlag 等编译情报。
2. 渲染器 `patch(n1, n2)` 按 ShapeFlags 分派：旧节点为空走挂载，存在则复用 DOM 走更新，各类型各走各的处理函数。
3. PatchFlag 加 dynamicChildren 构成 Block 快路径：更新只遍历动态节点平级数组，静态节点直接跳过。

## 自测一题

**问：** 为什么我手写 `h('div', [h('span', count)])`，明明只有文字在变，却拿不到 Block 优化、只能全量 diff？

**答：** Block 快路径是模板编译器专属的福利。`h()` 内部会先 `setBlockTracking(-1)` 暂停收集，产出的节点 `patchFlag` 为 0、`dynamicChildren` 为 null；渲染器见到没有 Block 数组，就走 `patchChildren` 全量 diff 兜底。只有编译产物 `openBlock()` 加 `createBlock()` 打出的 PatchFlag 与动态节点名单，才能让更新按下标靶向执行。

**再问：** 给编译出的节点手动 `cloneVNode` 追加一个动态属性，会发生什么？

**答：** 原编译器的优化承诺作废了，`cloneVNode` 会把 `patchFlag` 补上 `FULL_PROPS`，props 更新退回全量 diff。同理，CACHED（负 1）与 BAIL（负 2）是负值特殊标志，只能全等比较，绝不能按位与。
