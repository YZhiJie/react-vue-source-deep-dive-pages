# 漫画 · 第 12 话 响应式：Link 链表心脏

> 对应正文：[第 2 章 响应式系统：Vue 3.5 的 Link 链表心脏](../../docs/vue/02-reactivity.md) ｜ 原画：[02-reactivity.svg](./02-reactivity.svg)

## 登场角色

- **阿V**：Vue 绿餐厅的老板娘，这一话搬出店里的「Dep 门铃总控」，讲 3.5 换上的新心脏。
- **阿R**：React 餐厅来串门的蓝色机器人，还抱着 Vue 2 的旧门，一脸困惑。
- **码叔**：猫头鹰教授，片尾划重点：Link 链表、track 与 trigger、Proxy 代理。

## 剧情梗概

阿R 发现 Vue 店里房间之间不靠登记册（subs Set）联系：数据房间与每个 effect 小房间之间都拉着一条 Link 线，读数据时接线、写数据时沿线摇铃，线上订阅者当场亮灯。

最后阿R 还抱着 Vue 2 的 `Object.defineProperty` 旧门——后来新增的属性连门铃都没有；阿V 给他换上 Proxy 新门：整门代理，新增属性、数组下标都能响，`$set` 从此退场。

## 分格解读

### 第 1 格 · 门铃连线（Dep 与 Link）

数据房间挂着「Dep 门铃总控」的门牌，它和每个 effect 小房间之间都有一条 Link 线（L1、L2……）。响应式系统本质是一张二分图：左边是 Dep（数据源），右边是订阅者（effect、computed），谁订阅谁，中间就有一条边。

3.5 里这条边就是 Link 对象：它同时挂在两条双向链表上，沿 deps 链回答「我依赖了谁」，沿 subs 链回答「我被谁订阅」。一条边只分配一个对象，两个方向都能遍历，不再需要任何 Set 容器。

注意 targetMap 并没有消失：「对象到 key 再到 Dep」的索引仍是 WeakMap 到 Map，变的只是 Dep 内部装订阅者的方式。

### 第 2 格 · 读时连线，写时摇铃

effect 运行期间读取数据（get），Proxy 的 get 拦截器调 `track`：没有线就 new 一条 Link 接上；老线本轮还在用就原地复用（version 从 -1 同步为 dep 当前版本）。写入数据（set）时 `trigger` 先把 dep 的 version 加一，再沿 subs 链从尾到头通知，画中金线上的订阅者当场亮灯。

通知整体包在批处理里：同一个值在同步代码里连改一百次，只有第一次通过 NOTIFIED 检查进入队列，微任务中只冲刷一轮，拿到的是第一百次的最终值。computed 更省——上游变化只给它打 DIRTY 脏标记并沿图传播，此刻没有任何 getter 执行，等有人读取时才惰性求值。

### 第 3 格 · Proxy 整门代理

阿R 抱着的旧门是 Vue 2 的 `Object.defineProperty`：它只能逐个拦截定义时已存在的属性，后来新增的 key 没有 getter 与 setter，自然没有门铃，只能靠 `Vue.set`（`$set`）补救；数组下标与 length 也拦不住，要包装七个变更方法。

阿V 的新门是 Proxy：一次代理整个对象，get、set、has、deleteProperty、ownKeys 全包。新增 key 走 ADD 拦截，除触发该 key 的 Dep 还触发 ITERATE_KEY，所以连 `v-for` 枚举到新属性都会更新；数组新下标联动 length，Map 与 Set 则替换整套劫持方法。

深层对象还是惰性代理：访问到哪一层才 reactive 哪一层，初始化时不再递归遍历整个对象。

## 码叔划重点

1. Vue 3.5 用 Dep 与 Link 双向链表替代 subs Set：依赖不是登记表，而是 effect 与 dep 之间的一条边（Link），同挂 deps、subs 两条链。
2. get 时 track 建立或复用 Link，set 时 trigger 自增 version 并沿链通知；computed 只打脏标、惰性求值，批处理保证连改多次只更新一轮。
3. Proxy 一次代理整个对象，新增属性与数组下标自动响应，Map、Set 也支持，Vue 2 时代的 `$set` 与数组七补丁都不再需要。

## 自测一题

**问：** Vue 3.5 为什么用 Link 链表取代 3.4 的 `Set<effect>`？最直观的收益是什么？

**答：** Set 容器即使只有一个订阅者也常驻内存，effect 每轮重跑前还要全量 cleanup、跑完再加回，GC 压力大。Link 方案里每条依赖只是一个同时挂在两条链上的边对象：老边不删，只把 version 标成 -1，本轮没用到才摘除、用到就原地复用；订阅计数归零后 Dep 还能自动从 Map 注销，内存与增删开销都更小。

**追问：** 为什么 `obj.newKey = 1` 在 Vue 2 里界面不更新，Vue 3 里却会更新？

**答：** defineProperty 只能拦截定义时已存在的 key，新属性没有 setter，只能 `$set` 补救；Proxy 代理整个对象，新增 key 走 ADD 拦截，除触发该 key 的 Dep 外还触发 ITERATE_KEY，所以枚举这个对象的 `v-for` 也会收到通知。
