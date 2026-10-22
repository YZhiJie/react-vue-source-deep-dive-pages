# 漫画 · 第 17 话 编译器：给半成品贴标签

> 对应正文：[第 7 章 编译器与编译优化](../../docs/vue/07-compiler.md) ｜ 原画：[07-compiler.svg](./07-compiler.svg)

## 登场角色

- **阿V**：Vue 绿餐厅主厨，本话坐镇中央厨房，兼任流水线监工与质检员，厨师帽全程不摘。
- **阿R**：React 餐厅的蓝色机器人，又来串门，盯着冰箱和压板一脸不解。
- **码叔**：猫头鹰教授，每话结尾出来划重点。

## 剧情梗概

第 11 话留下一个悬念：运行时凭什么敢「只更新动态节点」？这一话阿V 带阿R 钻进中央厨房——模板字符串在打包时经过 **parse → transform → generate** 三段流水线，先切配成 AST，再由质检员给会变的节点贴上彩色 PatchFlag 标签，最后装盒成 render 函数；连静态食材也有冰箱、备菜盘、压板三套优化安排。

运行时拿到的已经是一盒贴满标签的半成品，只管照标签补菜，绝不现场分析。这就是 Vue 和 React 最大的路线分歧：React 的 JSX 编译后近乎无语义，分析全留给运行时 Fiber；Vue 把能提前算的都前移到了编译期。

## 分格解读

### 第 1 格 · 中央厨房三段流水线

一条传送带三个工位：**parse** 把模板字符流切配成 AST（tokenizer 状态机逐字符迁移，parser 组装成树）；**transform** 是最忙的腌制工位，用 DFS 遍历 AST，每个插件进入节点时先登记、退出时再逆序改写，PatchFlag、dynamicProps 都是在这里打上去的；**generate** 最后装盒，生成 import helpers、`_hoisted_` 声明和 render 函数的代码字符串。

主流工程里这一切在 Vite 构建期由 `@vitejs/plugin-vue` 跑完，模板直接变成普通 JS 模块，运行时零编译开销；只有完整版 vue 包才会在组件带 `template` 字符串时现场 `new Function` 编译，而且同一模板只编译一次，结果缓存在 compileCache 里。工程选型应优先 runtime-only 版本，包更小、启动更快。

### 第 2 格 · 动态节点贴标签

transform 质检员挨个检查节点：纯静态的 h1 什么都不贴；含插值的 p 贴上红标签 `TEXT = 1`，带动态 class 的 span 贴上橙标签 `CLASS = 2`（两者同时命中就是 3）。标签和动态 props 键名数组全部在**编译期**算成数字，挂牌上写得明白——运行时只看标签更新，不做任何分析。手写 render/h() 没有这道质检，所以走不了这条优化捷径。

常见标签数字还有 `4 STYLE`、`8 PROPS`（配合 dynamicProps 只比对动态键）、`128 KEYED_FRAGMENT`（keyed v-for 的入口）、`512 NEED_PATCH`（指令/ref）等。这套数字就是第 3、4 章渲染器 patch 分支与第 4 章 keyed diff 的消费契约：编译期贴标，运行时照单执行。

### 第 3 格 · 静态优化三件套

3.5 给静态食材安排了三件道具：**① _cache 冰箱**——静态 vnode 首次 render 时创建并放进 render 的 `_cache`，打 `-1 CACHED`，之后 diff 整块跳过（注意不再是旧版的 `_hoisted_` 变量提升）；**② 提前摆好的备菜盘**——静态 props 对象和 dynamicProps 键数组仍是不可变纯数据，继续提升到 render 之外；**③ 压板**——连续 20 个静态节点（或 5 个带绑定的静态元素）直接 stringify 成一块 `createStaticVNode` innerHTML，一次生成 DOM，连 vnode 对象都省了。

为什么从模块级提升改成 render 内缓存？旧方案复用同一个静态 vnode 对象，运行时挂载会把 DOM 引用写回去，热更新或多实例下容易状态串扰；`_cache` 方案首次 render 才创建，安全和性能兼得。另外 `v-once` 与 `v-memo` 也复用同一个 `_cache` 机制：前者只求值一次并退出 Block 收集，后者依赖数组浅比较相同就整棵子树跳过。

## 码叔划重点

1. 编译器三段：parse 出 AST，transform 用 DFS 加退出回调改写打标，generate 生成 render 代码字符串。
2. PatchFlag 与 dynamicProps 都是编译期产物，运行时只消费、不分析——这是 Vue「编译时知道一切」的契约。
3. 3.5 静态优化三件套：静态节点进 `_cache`、静态 props 提升、连续静态节点 stringify 成 innerHTML。

## 自测问答

**问 1：** transform 为什么要 DFS 进入时登记、退出时才逆序执行改写？

**答：** 因为父节点的 PatchFlag 取决于子节点的常量性，v-if/v-for 也要等子表达式成型后才能整体改写成三元和 `renderList`。先子后父逆序退出，保证插件改写父节点时所有子节点已经是最终形态。

**问 2：** 在 3.5 编译产物里看到 `_cache[0] || (_cache[0] = _createElementVNode(..., -1))`，说明这个节点经历了什么？

**答：** 它是纯静态节点，被 cacheStatic 优化：首次 render 创建 vnode 存入 `_cache` 并打 `CACHED(-1)`，之后复用缓存、patch 时整块跳过。3.5 里这已经取代了旧版 `_hoisted_` 变量提升。

**问 3：** 为什么工程里推荐用 runtime-only 版本而不是自带编译器的完整版？

**答：** 主流构建链在打包时就已经把 SFC 模板编译成 render 函数，运行时根本不需要编译器；runtime-only 版本因此包体积更小、启动更快，还能避免运行时 `new Function` 的编译开销。完整版只适合在页面里直接写 `template` 字符串、没有构建步骤的场景。
