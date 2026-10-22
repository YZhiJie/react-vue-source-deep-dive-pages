# 漫画 · 第 8 话 Vue Router：楼层导览与守卫闸机

> 对应正文：[第 8 章 Vue Router 路由](../../docs/vue/08-vue-router.md) ｜ 原画：[08-vue-router.svg](./08-vue-router.svg)

## 登场角色

- **阿V**：Vue 绿餐厅主厨，本话身兼三职——电梯口的楼层导览员、VIP 包间的闸机守卫长、大堂电子楼层牌演示员。
- **阿R**：React 餐厅的蓝色机器人，又来串门，盯着黑灯楼层一脸问号。
- **码叔**：猫头鹰教授，每话结尾出来划重点。

## 剧情梗概

阿V 的绿餐厅也是一栋多楼层大楼。客人报上门牌号（URL），导览员不靠「先到先得」，而是掏出**打分表**：静态门牌 80 分稳赢动态门牌 60 分，客人报 `/teams/new` 就被精准带到「新建队伍」房间，绝不会误闯「某个队伍 :id」——这就是 matcher 的 tokenize + PathScore 打分排序。

进 VIP 包间前要连过三道闸机：大堂总闸（beforeEach）、楼层闸（beforeEnter）、包间门口闸（beforeRouteEnter）。每道闸都有三种裁决——放行 `next()`、拦下 `next(false)`、改道 `next('/login')`，一环套一环的洋葱模型。

大堂里的电子楼层牌 `currentRoute` 是一颗 `shallowRef`：客人一换楼层，整牌翻新，所有看牌的组件（RouterView、面包屑、导航高亮）同步刷新，无需逐个通知。而有些楼层平时黑着灯，客人第一次按门铃时才临时喊装修队进场——路由懒加载的动态 import。

## 分格解读

### 第 1 格 · 楼层导览牌打分（matcher）

routes 配置在 `createRouterMatcher` 里被编译成一棵 matcher 树加一个**按分数排序的扁平数组**：`tokenizePath` 用五态状态机把路径切成段（segment），`tokensToParser` 每段从 40 起评——静态 token 加 40（共 80），动态参数加 20（共 60），自定义正则再加 10，`.*` 通配倒扣 50、可重复再扣 20。于是 `/teams/new` 得 `[80, 80]`，`/teams/:id` 得 `[80, 60]`，`/:pathMatch(.*)*` 直接负分垫底。

插入时 `insertMatcher` 用二分查找按分数定位，`resolve` 时 `matchers.find(m => m.re.test(path))` 取第一个命中者——**分数高的天然在前，与 routes 书写顺序无关**。这就是客人报 `/teams/new` 永远进「新建队伍」而非「:id 房间」的原因。两个方向互逆：name 导航走 `stringify`（params 拼路径），path 导航走 `parse`（正则提 params）。

### 第 2 格 · 三道闸机（守卫洋葱）

完整守卫链是六段 Promise 链：`beforeRouteLeave`（深层优先）→ 全局 `beforeEach` → `beforeRouteUpdate` → 路由独享 `beforeEnter` → 组件 `beforeRouteEnter` → 全局 `beforeResolve`，全部过了才 `finalizeNavigation`，最后 `afterEach`。漫画里画的是客人视角最直观的三道闸。

每道闸的裁决权在 `guardToPromiseFn` 手里：`next(false)` 映射 `NAVIGATION_ABORTED`（拦下）；`next('/login')` 或 `return '/login'` 映射 `NAVIGATION_GUARD_REDIRECT`（改道——`pushWithRedirect` 会带着 `redirectedFrom` 重走一遍完整管线）；`next()` 或无返回则 resolve 放行。`runGuardQueue` 用 reduce 把守卫数组串成 Promise 链，任一 reject 后续全部短路；每段末尾还检查 `pendingLocation`——守卫跑着跑着客人改主意去了别处（发起新导航），旧导航立即以 `NAVIGATION_CANCELLED` 作废，绝不覆盖新导航。

### 第 3 格 · 响应式楼层牌 + 黑灯楼层

`currentRoute` 是 `createRouter` 开头就建好的 `shallowRef`，初始值 `START_LOCATION_NORMALIZED`。导航确认后 `currentRoute.value = toLocation` **整体替换**——这就是「路由变化 → 组件重渲染」的扳机。`install` 时 provide 的 `reactiveRoute` 不是深响应式，而是对每个 key 定义 getter 转发到 `currentRoute.value[key]` 再包一层 `shallowReactive`：任何组件读 `route.params.id` 时都会 track 到这个 ref，所以翻牌瞬间 RouterView、面包屑、导航高亮全部自动刷新，`useRoute()` 拿到的就是它。

旁边的懒加载楼层：`component: () => import('./Detail.vue')` 时 record 里存的是函数，平时黑灯不加载；首次导航提取守卫时才调用它发起 chunk 请求（与守卫执行并行），resolve 后把 `record.components[name]` **改写为组件本体**——第二次再来就是同步直出，灯已常亮。chunk 加载失败会让 Promise reject 沿守卫链冒泡到 `router.onError`，业务里常捕获后强制整页刷新。

## 码叔划重点

1. matcher = tokenize 切段 + PathScore 打分 + 编译正则：静态 80 ＞ 动态 60 ＞ catchAll 负分，有序数组里第一个命中即最优，与书写顺序无关。
2. 守卫洋葱六段链：leave → beforeEach → update → beforeEnter → enter → beforeResolve；`next(false)` 拦下、`next(location)` 改道重走管线、`next()` 放行，段间检查取消。
3. `currentRoute` 是 `shallowRef`，`$route` / `useRoute()` 全靠 getter 转发收集依赖，翻牌即整体替换、依赖组件自动刷新；懒加载组件首次导航才请求 chunk 并缓存改写。

## 自测一题

**问：** routes 里把 `/teams/new` 写在 `/teams/:id` 后面，为什么访问 `/teams/new` 还是先进「新建队伍」页面？

**答：** matcher 不按书写顺序匹配。每条路由编译时按 PathScore 累计分数（静态段 80 ＞ 动态段 60），`insertMatcher` 用二分查找把 matcher 按分数从高到低插进有序数组；`resolve` 时 `find` 返回第一个正则命中者，分数高的 `/teams/new` 天然排在 `/teams/:id` 前面。同理 catchAll 负分垫底，404 路由写在哪都行。

## 进阶追问

**问：** 守卫里 `return '/login'` 之后发生了什么？为什么写不好会无限重定向？

**答：** `guardToPromiseFn` 把它映射成 `NAVIGATION_GUARD_REDIRECT`，这不是终态而是 `pushWithRedirect` 的递归燃料：捕获后把 `'/login'` 作为新目标**重走一遍完整守卫管线**，并把原目标记进 `redirectedFrom`。如果守卫没先判断 `to.path !== '/login'`，重导航又命中同一个守卫再 return 一遍，就死循环了——dev 下靠 `redirectedFrom._count` 超过 30 次兜底 reject，生产环境没有这个保险，redirect 条件必须自己保证收敛。

## 下一话预告

本话是 Vue 篇最后一话。从响应式、渲染器、diff、组件系统、组合式能力、编译器到路由，绿餐厅的家底全部逛完——下一话正面对决：**[第 18 话 React vs Vue：两位厨师的做菜哲学](../comparison/react-vs-vue.md)**，看两位主厨在擂台上亮出各自的源码底牌。
