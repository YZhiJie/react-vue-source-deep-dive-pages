# 漫画 · 第 11 话 React Router：分厅导航台

> 对应正文：[第 11 章 React Router 路由](../../docs/react/11-react-router.md) ｜ 原画：[11-react-router.svg](./11-react-router.svg)

## 登场角色

- **阿R**：React 餐厅主厨，本话兼任分厅导航员，手持路径打分表，带客人找对房间。
- **阿V**：隔壁 Vue 绿餐厅主厨兼老板娘，第一次来多楼层多包间的大餐厅，进门就走迷路了。
- **码叔**：猫头鹰教授，每话结尾出来划重点。

## 剧情梗概

阿R 餐厅扩建成多楼层多包间，阿V 一进门就迷路：「/teams/new 到底去哪间？」阿R 导航员掏出路径打分表——静态段 10 分、动态段 3 分、通配倒扣 2 分，/teams/new 以 23 分稳赢 /teams/:id 的 16 分，房间一找一个准。阿V 在候客区刚坐下，三辆备菜小车已并行出发：loader 先行，数据齐了 URL 才翻牌。最后目标包间客满，loader 递回一张 302 小票，导航台直接把目的地改到 /login 重走一遍，阿V 全程无感知。

## 分格解读

### 第 1 格 · 打分表找房间

阿V 的迷路对应 `matchRoutes` 要解决的问题：URL 来了，到底匹配哪条路由？v6 起 React Router 不再看书写顺序，而是 `flattenRoutes` 把嵌套路由拍平成分支、`computeScore` 给每条分支算分、`rankRouteBranches` 排序后逐个试配。打分规则就藏在黑板里：静态段 +10、动态段 `:id` +3、含 `*` 通配 -2，段数打底。所以 `/teams/new`（23 分）永远赢 `/teams/:id`（16 分），v5 时代「精确路由要写前面」的口诀彻底作废。同分时才比书写位置，兄弟索引靠前者优先。

### 第 2 格 · loader 先行备菜

阿V 还在候客区，菜已上路——这正是 v6.4 数据路由的核心时序：**导航先跑 loader，数据就绪才提交渲染**。`startNavigation` 匹配出整条 matches 链后，`defaultDataStrategy` 用 `Promise.all` 把所有 `shouldLoad` 的 loader 并行发出，三层嵌套路由的三个 loader 同时开跑，彻底告别「父组件 render 完才发子请求」的瀑布。代价是子 loader 不能依赖父 loader 的返回值。URL 翻牌发生在 `completeNavigation` 里——loader 没跑完前地址栏还是旧的，这就是「点了 Link 地址栏没变」不是 bug 的原因。

### 第 3 格 · redirect 小票改道

包间客满怎么办？loader 里 `throw redirect("/login")` 就行。`redirect()` 没有任何魔法，只是造一个带 `Location` 头、状态码 3xx 的标准 `Response`——画里那张「302 小票」。内核在 loader/action 返回后检查状态码，识别到 3xx 就让本次导航**短路作废**，再以小票上的 Location 为目的地递归调用 `startNavigation`，重新走一遍匹配-加载流程。客人（用户）只看到最终落座的房间，中间那次作废的导航毫无感知。推荐 `throw` 而非 `return`：能从深层工具函数直接终止调用栈。

## 码叔划重点

1. 匹配靠打分不靠书写顺序：`flattenRoutes` 拍平分支、`computeScore` 静态段 +10 / 动态段 +3 / 通配 -2，`/teams/new` 稳赢 `/teams/:id`。
2. loader 先行：`defaultDataStrategy` 用 Promise.all 并行跑所有 shouldLoad 的 loader，数据就绪后 `completeNavigation` 才改 URL、广播新 state——先备料再上菜。
3. redirect 只是一张 3xx 小票：内核识别 Location 头后短路当前导航、以新目的地重新导航，用户无感知。

## 自测一题

**问：** 为什么把 `/teams/:id` 写在路由表最前面，也抢不走 `/teams/new` 的客人？

**答：** 因为 v6 起的匹配器与书写顺序无关。`matchRoutes` 先用 `flattenRoutes` 深度优先把嵌套路由拍平成若干「分支」，再由 `computeScore` 给每条分支算分：段数打底，静态段 +10、动态段 +3、空段 +1、含 `*` 罚 -2、index 路由 +2。`/teams/new` 是 3 段 + 10×2 = 23 分，`/teams/:id` 是 3 段 + 10 + 3 = 16 分，`rankRouteBranches` 按总分从高到低排序后逐个 `matchRouteBranch` 试配，23 分的 `/teams/new` 永远先被试中。书写顺序只在**同分**时通过 `compareIndexes` 当决胜局——兄弟索引小的优先。所以 v5「精确路由放前面」的心智模型可以扔掉了。

## 进阶追问

**问：** 既然所有 loader 是并行跑的，子路由需要父路由的数据（比如父 loader 查出的 teamId 对应的权限）该怎么办？

**答：** 并行的代价正是子 loader 拿不到父 loader 的返回值——它们同时发出。三条出路：其一，把有依赖关系的数据合并到同一个 loader 里一次取回，子路由通过 `useRouteLoaderData` 或 matches 共享；其二，子 loader 从 `params`（URL 里就有 `:teamId`）或 request 自己重新取数，接受轻度重复；其三，用 v7 的 middleware / 自定义 `dataStrategy` 显式编排串行依赖。这也引出一个架构问题：不属于任何 URL 的全局状态（登录态、购物车）放 loader 里会随路由反复加载——那正是下一话 Redux 要管的「账」。

## 下一话预告

路由管的是「找房间」，可登录态、购物车这些全局账本该谁管？下话 EP.20 Redux 篇，阿R 捧出一本厚厚的中央账本——单一 store、dispatch 记账、reducer 算账，一笔一笔算给你看。
