# 漫画 · 第 5 话 协调与 Diff：key 事故现场

> 对应正文：[第 5 章 协调与 Diff 算法](../../docs/react/05-reconciliation-diff.md) ｜ 原画：[05-reconciliation-diff.svg](./05-reconciliation-diff.svg)

## 登场角色

- **阿R**：React 餐厅的服务员兼主厨，蓝色小机器人，胸口的「R」是它的骄傲。本话兼任「协调科」办事员，专门给新旧 Fiber 节点对号入座。
- **阿V**：隔壁 Vue 绿餐厅的主厨兼老板娘，绿发少女，头顶别着 Vue 同款 V 字发饰。这次以顾客身份撞上了 key 事故。
- **码叔**：猫头鹰教授，每话结尾出来划重点。

## 剧情梗概

阿V 的餐厅列表在**头部**插了个人，输入框里还没提交的文字竟然「串」到了别人碗里——一桩标准的 key 事故现场。阿R 一边擦汗一边办案：先承认「没写 key 只能按座位号复用节点」，再立起三大假设的牌子，讲清 React 凭什么决定复用还是重建。

最后阿R 掏出一本「key 名册」，演示乱序列表里的对号流程：旧节点按 key 建册，新列表逐个查号——序号还递增的就原地不动，序号穿越了游标的才打 Placement 搬家，查不到的统一删除。

## 分格解读

### 第 1 格 · 串台事故

对应正文 5.1 与 5.8.1：`{list.map((item, i) => <InputRow key={i} />)}` 用数组下标当 key，key 表达的是「它排在第几」，而不是「这条数据是谁」。

头部插入 X 后，key=0 的旧 fiber 依然命中（key 相同且 type 相同），React 走 `useFiber` 复用、只把 props 从 A 换成 X；而承载 A 数据那一行的内部 state（输入框文字）还留在现在显示 X 的组件实例上——状态整体错位一行。React 不是把字「搬错」了，是身份本来就按座位（位置）分配，state 跟着 fiber 实例走，不跟着数据走。

### 第 2 格 · 三大假设

对应正文 5.2：两棵树做精确最小转换是 O(n³) 的经典难题，React 不敢这么干，于是「认命」立牌：

- **类型不同 → 拆了重建**：`elementType` 不匹配就删旧 fiber、新建 fiber，state 全部丢失。
- **类型相同 → 只改属性**：`useFiber` 克隆旧 fiber，只换上新的 pendingProps，实例与 state 保留。
- **key 相同 → 就是同一个人**：key 决定列表中节点的身份，数组 Diff 全程先比 key 再比 type。

再配上「只同层比较、不跨层」这条底层前提，三大假设把复杂度从 O(n³) 压到 O(n)。React 选的是**简单、快、可预测**，而不是操作次数最少。

### 第 3 格 · 名册对号

对应正文 5.6 与 5.7：多节点 Diff 的策略是先捡便宜的、再处理乱序的，分两轮走。

第一轮 `updateSlot` 顺序对齐、撞 key 即 break；尾部追加能一路走到 `createChild` 快路径，完全不建 Map。第二轮把剩余旧 fiber 用 `mapRemainingChildren` 按 key 建成 Map（没 key 的用 index 当键），`updateFromMap` 逐个查册复用。

要不要搬家由 `placeChild` 用单调不减的 `lastPlacedIndex` 判定：**oldIndex 小于 lastPlacedIndex 才打 Placement**——序号还递增就不挪窝，序号穿越了游标的才移动；Map 里没被取走的旧节点最后统一进 deletions 删除。这套算法天然偏爱尾部变化，不追求最少移动次数。

## 码叔划重点

1. Diff 三假设把 O(n³) 优化到 O(n)：跨类型重建、同类型复用、key 自报家门。
   - 放弃最优解换可预测，这是 Fiber 无反向指针下的刻意工程取舍。
2. 不写 key 按位置复用会 state 串台；用数组下标当 key 等于没写。
   - key 要标识「这条数据是谁」，而不是「它排在第几」。
3. 乱序 diff：旧列表按 key 建 Map，遍历新列表比对，oldIndex 递增则不移动。
   - 游标 lastPlacedIndex 只增不减，记录已确认节点在旧序列中的最右位置。

## 自测一题

**问：** 同样是头部插入一条数据，为什么用数组下标当 key 会串台，换成稳定的业务 id 就不会？

**答：** 复用的判定条件是 key 且 type 双匹配。index 作 key 时，头部插入后 key=0、key=1 依然逐个命中旧 fiber，React 以为「同一个人」还在，只更新 props（A 换成 X、B 换成 A），旧 A 行的输入框文字作为内部 state 被留在了现在显示 X 的实例上，整体错位一行。稳定 id 下 key=A、key=B 的旧 fiber 仍能被正确找到，只是 `placeChild` 依据 lastPlacedIndex 给真正穿越了游标的节点打 Placement，state 跟随身份走，自然不串台。
