# 漫画 · 第 13 话 Zustand：公告栏与业主群

> 对应正文：[第 13 章 Zustand 状态管理](../../docs/react/13-zustand.md) ｜ 原画：[13-zustand.svg](./13-zustand.svg)

## 登场角色

- **阿R**：React 餐厅主厨，本话放下中央账房的架子，改用小区公告栏管状态。
- **阿V**：隔壁 Vue 绿餐厅主厨兼老板娘，看完 Redux 的全套流程有点晕，特来看看有没有轻一点的玩法。
- **码叔**：猫头鹰教授，每话结尾出来划重点。

## 剧情梗概

上一话 Redux 的「中央账本 + 柜员管线」虽然严谨，但 action type、reducer、Provider、useSelector 一样不能少，小本生意也要全套流程。这一话阿R 把账房撤了，挂出一块**公告栏白板**：state 锁在闭包里，谁想看就看，想改拿 `set` 马克笔直接写——没有 Provider、没有 Context、没有申请单。阿V 担心「谁都能改，怎么不乱？」阿R 再亮两件法宝：**业主群分群**（selector 只订自己那栏）和 **persist 保险柜**（写完顺手拍照存 localStorage，刷新后 hydration 贴回）。

## 分格解读

### 第 1 格 · 公告栏白板

阿V 惊讶「连 Provider 都不用包？Context 也不要？」阿R 举马克笔写白板：`create()` 返回的就是一个 hook，**不需要任何 Provider 包裹**——因为 vanilla 层的 `createStore` 用闭包把 `state` 和 `listeners` Set 锁死，模块加载那一刻就已经存在唯一一份，组件只是订阅这个全局单例而已。

白板上写着 `count: 0`、`user: 阿R`、`inc(): +1`——Zustand 的 store 同时装**数据**和**方法**，`set` 默认浅合并，写一行 `set({ count: get().count + 1 })` 就改完，再通知所有 listeners。60 行 vanilla + 66 行 react 胶水，就是全部。

### 第 2 格 · 业主群分群

阿R 举喇叭喊「count 栏更新啦！」，**只有订了 count 群的业主抬头**——user 群、theme 群继续睡。这就是 `useStore(s => s.count)` 的本质：每个组件用 selector 从整棵 state 树里挑自己关心的那一片，store 摇铃时用 `Object.is` 比对新旧选择结果，没变就跳过重渲染。

右下角紫色的 `useShallow` 尺是新手救星：selector 一旦返回对象或数组字面量（`s => ({ a: s.a, b: s.b })`），每次摇铃都是新引用，`Object.is` 永远 false、组件必重渲染。`useShallow` 内部用一个 ref 记住上次的返回值，对结果做**一层浅比较**，所有键的引用都没变就复用上次的引用，骗过 `Object.is`。

### 第 3 格 · persist 保险柜与 hydration

阿R 把白板拍照塞进 `localStorage` 保险柜，页面刷新后再取出来贴回去。整个过程由 `persist` 中间件完成：它包在 `create` 最外层，劫持 `set`——每次写完白板顺手 `JSON.stringify` 一份塞进 storage；启动时先同步摆出一份初始 state 让组件能渲染，再异步 `getItem` 做 **hydration**，把存的字段合并回去。

红剪刀 `partialize` 是安全细节：白板上的 `tmpDraft` 这类临时字段不该入库，`partialize: s => ({ count: s.count, user: s.user })` 显式挑要持久化的白名单，刷新后 draft 自然就「没了」。SSR 场景还可以用 `skipHydration: true` 把贴回的动作推迟到客户端手动触发。

## 码叔划重点

1. `create()` 一块白板：state 锁在 vanilla 闭包，`set` 浅合并 + 通知 listeners，**无 Provider 无 Context**，这是它和 Redux 最本质的区别。
2. selector 订栏目 + `Object.is` 浅比较；selector 返回对象/数组记得套 `useShallow`，否则每次摇铃都重渲染。
3. `persist` 中间件包装 `set`：写完拍照存 storage，刷新走 hydration 贴回；`partialize` 挑字段、`skipHydration` 控时序。

## 自测一题

**问：** 阿R 写了个 selector：`useStore(s => ({ count: s.count, user: s.user }))`，结果组件每次任意 state 变化都重渲染，明明 count 和 user 都没变，问题出在哪？

**答：** selector 每次执行都返回**新对象字面量**，`Object.is(旧结果, 新结果)` 永远为 false，`useStore` 认为选中值变了就触发重渲染——哪怕里面的字段一个都没变。两种修法：一是拆成两次 `useStore(s => s.count)` / `useStore(s => s.user)`；二是套 `useShallow`，让它对结果做一层浅比较，字段引用都没变就复用旧引用。

## 进阶追问

既然 Zustand 不用 Provider、直接订阅模块级单例，那**多个独立实例**（比如每个表单一个 store）和**服务端渲染**（每个请求一份 state）怎么办？提示：想想 `createStore` 这个底层 API 暴露了哪些钩子，配合 React 的 `useContext` 能不能手工再造一个「可选的 Provider」——这正是 zustand 官方 docs 里 `createContext + useStore` 方案的思路。

## 下一话预告

React 这边状态管理收了尾，隔壁阿V 也该带我们看她的路由了——Vue Router 怎么用 matcher 给路径打分？守卫为什么能像洋葱一样层层放行？下一话（EP.22 · Vue Router：楼层导览与守卫闸机）绿餐厅的楼层导览系统登场。
