# 漫画 · 第 16 话 Composition 与内置组件百宝箱

> 对应正文：[第 6 章 Composition API 与内置能力](../../docs/vue/06-composition-api.md) ｜ 原画：[06-composition-api.svg](./06-composition-api.svg)

## 登场角色

- **阿V**：Vue 绿餐厅的主厨兼老板娘，绿发少女。这一话她打开 Composition 百宝箱，把三层家底挨个给客人看。
- **阿R**：React 餐厅的蓝色机器人服务员，常来串门。看到一根长杆隔了三层还能从宝箱里取东西，眼睛都瞪圆了。
- **码叔**：猫头鹰教授，每话结尾出来划重点，这一话还带了 LRU 号码牌。

## 剧情梗概

百宝箱第一层是**生命周期钩子**：每个组件实例上有一排 14 个短名抽屉，回调卡片得趁 setup 同步执行期间塞进去，晚了不收。

第二层是 **provide / inject**：祖先把东西放进顶层宝箱，子孙拿一根长杆沿原型链往下接，隔多少层都能取到，东西本身（包括 ref 的响应式）一点不变。

第三层是 **KeepAlive 储物柜**：切走的组件不销毁，贴上名签睡进柜子，切回时原样唤醒；柜子满了按 LRU 规矩，请最久没被翻牌子的那位真正离开。

## 分格解读

### 第 1 格 · 钩子槽位（injectHook）

`onMounted`、`onBeforeMount`、`onUnmounted` 这些 API 长相各异，内核却只有一个 `injectHook`：把回调包一层统一错误处理，再 push 进实例上的短名字段。

- 实例上一共 14 个槽位：`bc`、`c`、`bm`、`m`、`bu`、`u`、`bum`、`um`、`da`、`a`、`rtg`、`rtc`、`ec`、`sp`，每个槽位都是**数组**，组合函数和 mixin 都往里 push，不用去重。
- `injectHook` 默认写入目标是全局的 `currentInstance`，它只在 setup 同步执行期间被设置——这就是抽屉上「只能 setup 同步期塞」牌子的全部含义；第一个 `await` 之后或事件回调里再注册，只会收到警告。
- setup 每个组件只执行一次，没有 React Hooks「反复 render 按顺序对齐」的约束，钩子写在 `if` 分支里完全合法。

### 第 2 格 · 依赖注入（provide / inject）

子实例创建时直接引用父级的 `provides` 对象；当这一层第一次调用 `provide` 时，才用 `Object.create(parentProvides)` 分叉出一个新对象。

- 分叉后本层写入不会污染父层；祖先层的 key 靠**原型链**天然可见，所以阿R 那根长杆隔了子、孙、曾孙三层照样能取到宝箱。
- `inject` 读的是 `instance.parent.provides`，方向不会搞反；根组件回落 `appContext.provides`，所以 `app.provide()` 全局可见。
- 注入只搬运引用、**不解包**：祖先 provide 的是 `ref`，后代 inject 到的就是同一个响应式对象，render 里读取照样建立依赖。想禁止后代改值，自己包一层 `readonly` 即可。

### 第 3 格 · KeepAlive 储物柜

KeepAlive 自己是个有状态组件，setup 闭包里维护 `cache: Map`（key 到 VNode）、`keys: Set`（访问次序）和一个游离 div `storageContainer`。

- 切走时走 deactivate：DOM 整块 move 进离屏容器，实例、响应式 effect、滚动状态全活着，触发 `deactivated` 进入休眠。
- 切回时走 activate：DOM 搬回页面，先 patch 一遍离开期间可能变化的 props，再触发 `activated` 唤醒。`onMounted` 只触发一次，每次进入都要跑的逻辑得放 `onActivated`。
- 缓存数超过 `max` 时，取 Set 迭代器头部（最久未用）那条记录真正 unmount——这就是格口被请出去的 D 签组件。
- 旁边的 Suspense 是同款离屏思路：default 子树先挂到 `hiddenContainer`，异步依赖 deps 计数归零后再整体 move 进真实容器，期间显示 fallback。

## 码叔划重点

1. `onMounted` 等钩子经 `injectHook` 写入实例短名字段（14 个槽位、数组存储），仅 setup 同步执行期间可注册。
2. provide / inject 靠「默认引用父级 provides、首次 provide 时原型链分叉」跨层传递，注入的 ref 依然响应式。
3. KeepAlive = Map 加 Set 的 LRU 缓存加离屏容器，休眠唤醒对应 deactivated / activated；Suspense 走 hiddenContainer 两阶段挂载。

## 自测一题

**问：** 为什么 `onMounted` 只能在 setup 同步执行期间调用，而 provide 进来的 ref 过了十层组件还能响应式更新？

**答：** `injectHook` 默认拿全局 `currentInstance` 当写入目标，它只在 setup 同步执行期间存在；第一个 `await` 之后或在事件回调里，实例上下文已经释放，target 为 null 自然注册不上。响应式则是 ref 自身的能力：provide / inject 全程只复制对象引用、从不解包，十层之后 inject 到的仍是祖先手里那同一个 ref，render 读取它时照样建立依赖，所以值一改就能精准通知到这个后代组件。
