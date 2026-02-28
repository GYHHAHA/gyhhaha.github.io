# Redux

## Redux 核心概念

### 1. 核心三大件：数据、指令与处理器

| 概念   | 面试金句（定义）                                              | 大白话理解                                              |
| ------ | ------------------------------------------------------------- | ------------------------------------------------------- |
| Store  | 保存程序所有状态的单一实体，是唯一的真理来源。                | 整个公司的“中央仓库”。                                  |
| State  | Store 中存储的具体数据，通常是一个大的 JSON 对象。            | 仓库里货架上的“货物清单”。                              |
| Action | 描述“发生了什么”的普通对象。必须包含 `type`，可选 `payload`。 | 一张“出库单”：`type` 是操作名称，`payload` 是具体包裹。 |

### 2. 核心逻辑：Reducer 与 Dispatch

#### Reducer (纯函数)

这是面试官最喜欢问的地方。

- 定义：`(previousState, action) => newState`。它根据旧状态和动作，计算出新状态。
- 三大原则：

1. 不得修改参数（不可变性）。
2. 不得执行副作用（禁止 API 请求、路由跳转）。
3. 不得调用非纯函数（禁止 `Date.now()` 或 `Math.random()`）。

> 面试提醒：为什么要纯函数？为了可预测性和时间旅行调试（Time Travel Debugging）。

#### Dispatch & Subscribe

- Dispatch：这是改变 State 的唯一途径。你把 Action 喂给 `dispatch(action)`，它内部会传给 Reducer。
- Subscribe：注册一个监听回调。每当 State 变化时，Reducer 执行完，Store 就会通知所有订阅者（通常是 UI 组件）。

### 3. 底层脉络：单向数据流 (One-way Data Flow)

这是 Redux 的灵魂。面试时如果能边画图边描述这个闭环，基本稳了：

1. Trigger (View)：用户点击按钮，产生一个意图。
2. Dispatch (Action)：调用 `store.dispatch({ type: 'ADD_TODO', payload: '学习Redux' })`。
3. Execute (Reducer)：Store 自动调用 Reducer，传入当前 State 和这个 Action。Reducer 返回全新的 State。
4. Update (Store)：Store 用新 State 替换旧 State，并触发 Subscribe。
5. Re-render (View)：UI 组件感知到 State 变化，获取新数据并重新渲染。

### 面试突击小贴士

- 为什么要不可变性 (Immutability)？
  Redux 通过比较对象的引用地址（`prevProps === nextProps`）来决定是否刷新。如果你直接修改旧 State 的属性，引用没变，React 可能就不会触发重渲染。
- 为什么叫 Redux？
  它是 Reducer + Flux 的结合。

## Redux Toolkit

### 1. `configureStore`：开箱即用的指挥中心

在传统 Redux 中，你需要手动组合 `combineReducers`、应用 `applyMiddleware`、配置 `DevTools`。

- 面试点：它比原来的 `createStore` 多做了什么？
- 答案：

1. 自动合并：传入 `reducer` 对象，它会自动调用 `combineReducers`。
2. 内置中间件：默认引入了 `redux-thunk`（处理异步）和检查常规错误的中间件（如：检测是否在 Reducer 中直接修改了 State）。
3. 开发工具：自动开启 Redux DevTools 扩展支持。

### 2. `createSlice`：Reducer 与 Action 的“合体”

这是 RTK 的灵魂，彻底解决了“样板代码”繁琐的问题。

- 核心机制：
- 自动生成 Action：你定义的函数名，就是生成的 Action Type。
- 内置 Immer：底层使用了 Immer.js。你可以在代码里写 `state.value += 1` 这种看起来像“直接修改”的代码，但 Immer 会拦截并将其转化为“不可变更新”。

- `reducers` vs `extraReducers`：
- `reducers`：定义该 Slice 内部生成的 Action 处理器。
- `extraReducers`：处理外部定义的 Action。比如处理 `createAsyncThunk` 生成的异步 Action，或者其他 Slice 的 Action。

### 3. `createAsyncThunk`：异步请求的标准化

以前写异步要手写 `REQUEST_START`、`SUCCESS`、`FAILURE` 三个常量，现在 RTK 帮你管。

- 三个状态（Promise Lifecycle）：

1. `pending`：请求中（通常在这里设置 `loading = true`）。
2. `fulfilled`：请求成功（拿到 `payload` 更新数据）。
3. `rejected`：请求失败（处理 `error`）。

- 面试金句：`createAsyncThunk` 本身并不发送请求，它只是一个Action 生产者，负责根据 Promise 的状态派发对应的 Action。

### 4. 高阶加分项：性能与扩展

#### `createEntityAdapter` (规范化数据)

- 痛点：如果你的 State 是一个数组 `[{id: 1, ...}, ...]`，更新某个 ID 的数据需要 $O(n)$ 时间复杂度。
- 方案：它将数据转化成 `{ ids: [], entities: {} }` 的哈希表结构。
- 面试话术：“它将列表操作转化为了 $O(1)$ 的查找效率，并提供了内置的 CRUD 方法（如 `addMany`, `upsertOne`），极大地简化了复杂列表的管理。”

#### Listener Middleware

- 用途：它是 RTK 1.8+ 引入的轻量级中间件，旨在取代繁琐的 `redux-saga`。
- 场景：当 A 动作发生时，我想触发 B 逻辑（比如：登录成功后，自动把 Token 存入 localStorage）。它比 Thunk 更解耦。

### ⚡️ 避坑指南（面试常考）

1. 问：RTK 允许直接修改 State 吗？

- 答：不允许。 只是因为 `createSlice` 内部集成了 Immer，让我们能用“突变”的语法写出“不可变”的逻辑。在 `createSlice` 之外，依然要严守不可变性原则。

2. 问：为什么还要用 `extraReducers`？

- 答：为了实现 “一个 Action 引起多个 Slice 响应”。比如 `logout` 动作触发时，所有的 Slice 都需要在 `extraReducers` 里监听它来重置自己的数据。

这一部分是面试官考察你“实战经验”的分水岭。如果说 Redux 是大脑，React-Redux 就是神经系统。

## React-Redux 集成

### 1. 连接层：Provider 与 Hooks

#### `Provider`：全局注入

- 作用：底层利用了 React Context API。它将 Redux Store 注入到整个应用的组件树中，使得任何层级的子组件都能访问到 Store。
- 面试点：为什么不直接在组件里 `import store`？
- 答案：为了解耦和可测试性。`Provider` 允许我们在测试时轻松更换 Mock Store，且符合 React 的声明式编程范式。

#### `useSelector` 与 `useDispatch`

- `useDispatch`：返回 Store 的 `dispatch` 方法，用于发送 Action。
- `useSelector`：
- 作用：从 State 中提取数据。
- 订阅机制：这是重点！`useSelector` 会自动订阅 Redux Store。每当 Action 被派发，它会运行选择器函数。如果返回的结果与上次引用不同，组件就会重渲染。

### 2. 性能优化：Selector 的艺术

这是高频考点，核心围绕 “避免无谓重渲染”。

#### 为什么 Selector 要尽量小？

如果你写 `const state = useSelector(state => state)`，那么 Store 中任何细微的变化都会导致该组件重渲染。

- 金句：“按需取值，粒度越细越好。” 只取组件真正需要的那个字段。

#### 避免返回“新对象”

如果在 `useSelector` 里写了如下代码，会导致组件每次 Action 派发时都重渲染（即使数据没变）：

```javascript
// ❌ 错误示范：每次都返回一个新数组引用
const userIds = useSelector((state) => state.users.map((u) => u.id));
```

- 解决方法：使用 `reselect` (RTK 已内置 `createSelector`)。

#### Memoization (Reselect)

`createSelector` 会缓存输入和输出。只有当输入参数（依赖的 State）发生变化时，它才会重新计算。

### 3. State 结构设计（架构思维）

#### 扁平化与规范化 (Normalization)

面试官可能会问：“如果你的数据嵌套了三层，怎么更新最快？”

- 痛点：深层嵌套会导致 Reducer 代码极其复杂（展开运算符 `...` 写到手软），且难以局部更新。
- 方案：像数据库一样建模。用 ID 作为键（Key），将数据存在 `entities` 对象中。
- 优势：

1. 更新快：修改某个 ID 的数据不影响其他 ID 的引用。
2. 查询快：$O(1)$ 时间复杂度直接定位数据。
3. 减少渲染：列表组件只传 ID 给子组件，子组件根据 ID 自行订阅，避免整个列表重绘。

### ⚡️ 面试实战 Q&A

Q: `useSelector` 是如何判断数据是否变化的？
A: 默认使用 `===` 严格相等检查。如果返回的是对象或数组，即便内容一样但引用换了，也会触发渲染。如果非要返回新对象，可以使用 `shallowEqual` 作为 `useSelector` 的第二个参数。

Q: 如果一个页面有海量数据，怎么优化 Redux 性能？
A: 1. 使用 `createEntityAdapter` 规范化数据。2. 使用 `createSelector` 进行记忆化查询。3. 确保 `useSelector` 的粒度足够小。4. 在父组件只传递 ID，由子组件独立连接 Store。

这是面试中的“压轴题”。很多候选人会说“Redux 功能更强”，但这太笼统。你需要从数据流向、性能优化、调试工具、适用场景四个维度来精准打击。

## Redux vs. Context API

| 维度        | Context API                                      | Redux (RTK)                                                 |
| ----------- | ------------------------------------------------ | ----------------------------------------------------------- |
| 定位        | 依赖注入工具。                                   | 状态管理框架。                                              |
| 数据流      | 简单的“提供者-消费者”模式。                      | 严格的“Action-Reducer-Store”单向流。                        |
| 性能        | 瓶颈： Provider 值变化，所有消费组件强制重渲染。 | 优势： `useSelector` 实现了原子级订阅，只有数据变化才重绘。 |
| 调试        | 依赖 React DevTools，难以追踪历史变化。          | 神器： Redux DevTools 支持“时间旅行”、状态快照。            |
| 异步/中间件 | 需配合 `useEffect` 或第三方库，无标准。          | 完善的中间件机制（Thunk, Saga, Listeners）。                |

### 1. 核心差异：重渲染机制（面试高频）

这是面试官最想听到的技术细节：

- Context API 的痛点：
  Context 本身并不具备“按需更新”的能力。如果你的 Context 存了一个包含 `user` 和 `theme` 的对象，当 `theme` 改变时，所有使用了 `user` 的组件也会跟着重渲染，因为它们订阅的是同一个 Context 对象。
- _方案：_ 必须拆分多个 Provider，或者手动加 `useMemo`。

- Redux 的优势：
  Redux 的 `useSelector` 内部做了引用对比检查。Store 是一个大的对象，但组件只“钩住”它需要的那一小部分。只要那一小块没变，组件就稳如泰山，绝不触发渲染。

### 2. 什么时候用 Context？什么时候用 Redux？

#### 场景 A：使用 Context API

- 数据更新频率极低（如：语言切换、主题颜色、用户信息）。
- 应用规模较小，不想引入额外的库体积。
- 主要是为了解决 Prop Drilling（属性透传） 问题。

#### 场景 B：使用 Redux (RTK)

- 高频更新的数据（如：实时股票价格、复杂表单、协作编辑）。
- 有复杂的业务逻辑，需要把逻辑从组件中抽离（写在 Thunks 或 Reducers 里）。
- 需要多人协作，Redux 强制的规范能保证代码风格统一，且调试方便。

### 3. 常见误区：有了 Hooks 还需要 Redux 吗？

面试官问： “`useReducer` + `useContext` 不就是 Redux 吗？”
你的回答：
“它们确实能模拟 Redux 的模式，但它们不是 Redux 的替代品。

1. 缺少中间件：`useReducer` 没法直接处理异步逻辑。
2. 性能短板：Context 的全量渲染问题在大型应用中是致命的。
3. 开发体验：Redux 拥有极其强大的插件生态（如持久化缓存 `redux-persist`）和 DevTools 调试能力，这是原生 Hooks 组合无法比拟的。”
