# 源码漫画剧场 · React × Vue 18 话

> 用 18 话分格漫画讲透 React 19.3.0 与 Vue 3.5.43 的内核。每话一幅原创 SVG 漫画（3 格剧情 + 码叔知识条）配一篇讲解，与 [../docs](../docs) 下 18 章正文一一对应：先看漫画建立直觉，再回正文啃源码。

## 登场角色

| 角色 | 身份 | 配色 |
| --- | --- | --- |
| **阿R** | React 餐厅主厨，蓝色小机器人，信条是「订单来了重算整桌菜」 | React 蓝 `#61dafb` |
| **阿V** | Vue 绿餐厅主厨，绿发少女，信条是「数据一动摇铃，只重做一盘」 | Vue 绿 `#42b883` |
| **码叔** | 猫头鹰教授，每话结尾出来「划重点」，专治只看剧情不记知识点 | 琥珀黄 `#f59e0b` |

## 怎么看

- **零基础热身**：按 EP.01 → EP.18 顺序看，每话 3 分钟，看完再读对应正文章节。
- **复习冲刺**：只看每话底部「码叔划重点」三条和讲解末尾的自测题。
- 所有漫画均为手写 SVG（统一 1200×800 画幅），在 GitHub / VS Code / 浏览器中直接打开即可，无需任何依赖。

## React 篇（EP.01 - EP.10）

| 话数 | 标题 | 漫画 | 讲解 | 对应正文 |
| --- | --- | --- | --- | --- |
| EP.01 | 架构总览：餐厅后厨流水线 | [SVG](react/01-architecture-overview.svg) | [讲解](react/01-architecture-overview.md) | [第 1 章](../docs/react/01-architecture-overview.md) |
| EP.02 | Element 与 Fiber：剧本与演出档案 | [SVG](react/02-element-and-fiber.svg) | [讲解](react/02-element-and-fiber.md) | [第 2 章](../docs/react/02-element-and-fiber.md) |
| EP.03 | Scheduler 与 Lane：急诊分诊台 | [SVG](react/03-scheduler-and-lanes.svg) | [讲解](react/03-scheduler-and-lanes.md) | [第 3 章](../docs/react/03-scheduler-and-lanes.md) |
| EP.04 | render 阶段：可中断的深度优先遍历 | [SVG](react/04-render-phase.svg) | [讲解](react/04-render-phase.md) | [第 4 章](../docs/react/04-render-phase.md) |
| EP.05 | 协调与 Diff：key 事故现场 | [SVG](react/05-reconciliation-diff.svg) | [讲解](react/05-reconciliation-diff.md) | [第 5 章](../docs/react/05-reconciliation-diff.md) |
| EP.06 | 状态更新链路：setN 的奇幻漂流 | [SVG](react/06-state-update-chain.svg) | [讲解](react/06-state-update-chain.md) | [第 6 章](../docs/react/06-state-update-chain.md) |
| EP.07 | Hooks：只认顺序的钩子链 | [SVG](react/07-hooks.svg) | [讲解](react/07-hooks.md) | [第 7 章](../docs/react/07-hooks.md) |
| EP.08 | commit：不可打断的上菜时刻 | [SVG](react/08-commit-and-lifecycle.svg) | [讲解](react/08-commit-and-lifecycle.md) | [第 8 章](../docs/react/08-commit-and-lifecycle.md) |
| EP.09 | 渲染器与事件：三层楼与总接待台 | [SVG](react/09-renderer-and-events.svg) | [讲解](react/09-renderer-and-events.md) | [第 9 章](../docs/react/09-renderer-and-events.md) |
| EP.10 | 并发与 Suspense：能放下的半成品 | [SVG](react/10-concurrent-and-suspense.svg) | [讲解](react/10-concurrent-and-suspense.md) | [第 10 章](../docs/react/10-concurrent-and-suspense.md) |

## Vue 篇（EP.11 - EP.17）

| 话数 | 标题 | 漫画 | 讲解 | 对应正文 |
| --- | --- | --- | --- | --- |
| EP.11 | 架构总览：中央厨房与传菜铃 | [SVG](vue/01-architecture.svg) | [讲解](vue/01-architecture.md) | [第 1 章](../docs/vue/01-architecture.md) |
| EP.12 | 响应式：Link 链表心脏 | [SVG](vue/02-reactivity.svg) | [讲解](vue/02-reactivity.md) | [第 2 章](../docs/vue/02-reactivity.md) |
| EP.13 | VNode 与渲染器：拼一桌模型 | [SVG](vue/03-vnode-and-renderer.svg) | [讲解](vue/03-vnode-and-renderer.md) | [第 3 章](../docs/vue/03-vnode-and-renderer.md) |
| EP.14 | Diff：为什么必须写 key | [SVG](vue/04-diff-algorithm.svg) | [讲解](vue/04-diff-algorithm.md) | [第 4 章](../docs/vue/04-diff-algorithm.md) |
| EP.15 | 组件的一生：从 setup 到 DOM | [SVG](vue/05-component-system.svg) | [讲解](vue/05-component-system.md) | [第 5 章](../docs/vue/05-component-system.md) |
| EP.16 | Composition 与内置组件百宝箱 | [SVG](vue/06-composition-api.svg) | [讲解](vue/06-composition-api.md) | [第 6 章](../docs/vue/06-composition-api.md) |
| EP.17 | 编译器：给半成品贴标签 | [SVG](vue/07-compiler.svg) | [讲解](vue/07-compiler.md) | [第 7 章](../docs/vue/07-compiler.md) |

## 终章对决（EP.18）

| 话数 | 标题 | 漫画 | 讲解 | 对应正文 |
| --- | --- | --- | --- | --- |
| EP.18 | React vs Vue：两位厨师的做菜哲学 | [SVG](comparison/react-vs-vue.svg) | [讲解](comparison/react-vs-vue.md) | [对比章](../docs/comparison/react-vs-vue.md) |

## 每话固定结构

```text
顶部横幅  系列名 + 话数标题 + EP 徽章
第 1 格   痛点/事故现场（先让你想起自己踩过的坑）
第 2 格   核心机制登场（数据结构/流程，术语与正文严格一致）
第 3 格   机制落地或对比收束
底部黑条  码叔划重点：三条必须带走的结论
讲解 MD   剧情梗概 → 分格解读 → 码叔划重点 → 自测一题（含答案与追问）
```

## 质量校验

零依赖 Node 脚本，校验 manifest 与磁盘双向一致、SVG 良构（`xmllint`）、画幅统一、可见文本无裸 XML 字符、讲解 MD 结构与正文链接：

```bash
node comics/check-comics.mjs
```

全绿输出 `PASS 18 话 ... 全部通过`；新增漫画时需同步登记 [manifest.json](manifest.json)。
