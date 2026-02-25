# React

## 如何理解虚拟DOM？

虚拟 DOM 本质上是 真实 DOM 的一种轻量级 JavaScript 对象抽象，在 React 中表现为普通的 Plain Object。当你编写 JSX 时，Babel 插件会将其编译为 React.createElement 的调用，执行后便在内存中生成了包含 tag、props 和 children 等属性的树形结构。这种设计将繁琐的原生 DOM 操作抽象化，使得开发者可以从“如何操作 DOM”转向“UI 应该是什么状态”，在提升代码抽象能力的同时，也有效规避了直接操作 DOM 带来的安全风险与高昂的维护成本。

从运行机制来看，虚拟 DOM 是 调和（Reconciliation）与 Patch 补丁算法的基石。每当组件状态变更，React 都会在内存中生成一棵新的虚拟 DOM 树，并将其与旧树进行 Diff 比较。计算出的差异被转化为 Patch（补丁） 任务，由渲染器一次性、批量地应用到真实 DOM 上。这种“计算差异后统一同步”的模式，不仅大幅减少了浏览器重排与重绘的次数，更赋予了 React 跨平台的能力——通过将虚拟 DOM 映射到不同的渲染引擎，React 可以轻松支持 Web、Native 移动端甚至 Canvas 绘图，实现了“一次学习，随处编写”的开发范式。

## 使用函数式之后，React组件还有生命周期吗？

使用函数组件之后，React 不再提供像 class 组件那样显式的生命周期方法（例如 componentDidMount、componentDidUpdate、componentWillUnmount），但生命周期的概念依然存在，只是通过 Hooks 来表达，比如 useEffect 可以同时处理“挂载、更新和卸载”阶段的逻辑（通过依赖数组控制触发时机，并通过返回函数实现卸载清理），useLayoutEffect 对应的是同步执行阶段，因此可以理解为：函数组件没有生命周期方法，但生命周期行为依然存在，只是被 Hooks 以函数式的方式组合和管理。

## 什么是Fiber架构？

在旧架构中，React 采用递归方式进行组件 diff，一旦开始更新就必须一口气执行完，期间无法打断，这在组件树很大时会阻塞浏览器主线程。

Fiber 的核心改进是：把一次耗时的更新任务拆分成很多小任务（时间切片），每做完一小段就把控制权交还给浏览器，如果有更高优先级任务（比如用户输入），可以先处理高优任务，从而避免卡顿。

- 任务碎片化（增量渲染）：把渲染工作拆成多个小单元，每个 Fiber 节点代表一个组件单元，逐个处理。
- 可中断与优先级机制：更新过程可以被打断；如果出现更高优先级的更新，当前低优任务会被丢弃，之后再重新执行。
- 调度能力（Scheduler）：为不同更新分配优先级（如用户交互 > 网络数据更新），提升响应速度。

Fiber 架构下的更新有两个阶段：

Reconciliation Phase（协调阶段）：这个阶段主要做“计算工作”：根据最新的 state 和 props 构建新的 Fiber 树，对比新旧 Fiber，找出哪些组件需要更新，以及最终哪些 DOM 需要变更；整个过程只是在内存中进行 diff 和标记副作用，不会真正操作 DOM，而且在 Fiber 架构下这一阶段是可中断、可恢复、可重启的，如果有更高优先级任务进来，当前任务可以被打断甚至丢弃后重来。

Commit Phase（提交阶段）：这个阶段会把前面标记好的变更一次性同步应用到真实 DOM，并执行相关副作用，比如更新 ref、执行 useLayoutEffect 和 useEffect 等；由于这一阶段会直接影响真实界面，因此必须是同步且不可中断的，以保证 UI 状态的一致性和稳定性。

## 什么是React UI更新的3个阶段？

整体设计是“render 可中断，commit 必须同步”，从而兼顾性能与稳定性。

- render 阶段主要根据当前 state 和 props 生成新的虚拟 DOM（在函数组件中就是执行组件函数），这一阶段是纯计算、无副作用的，并且在 Fiber 架构下可以被打断、重启或丢弃
- pre-commit 阶段属于提交前的准备阶段，此时已经确定了要变更的内容，可以安全读取真实 DOM（例如 getSnapshotBeforeUpdate 对应的时机）
- commit 阶段则是将变更真正同步应用到真实 DOM，并执行副作用的阶段，比如触发 ref 回调、执行 useLayoutEffect 和 useEffect 等，这一阶段必须同步完成以保证界面一致性

## setstate的过程和原理是什么？

在 React 中，setState 并不是一个简单的变量赋值，而是一个由 React 调度系统（Scheduler）管控的异步更新过程。当你调用 setState 时，React 会首先为当前更新分配一个优先级，并将这个更新包装成一个 Update 对象，推入该组件对应的更新队列中。随后，React 进入调度阶段，它并不会立即执行更新，而是根据当前浏览器的空闲程度和任务优先级（如点击事件比数据请求优先级更高）来决定何时触发渲染，这种机制保证了在大量状态变化时，浏览器依然能保持流畅的交互响应。

进入调和阶段（Reconciliation）后，React 开始构建新的 Fiber 树。它会根据新的状态运行组件的 render 方法，生成一份新的虚拟 DOM 树，并将其与当前的旧虚拟 DOM 树进行深度对比（即 Diff 算法）。在 Fiber 架构下，这个过程是可中断的，React 可以利用碎片时间进行计算。通过对比，React 能精准地找出哪些节点发生了变化，并将这些变更记录在 Fiber 节点的 effectTag 上，形成一个“副作用清单”。

最后是提交阶段（Commit），这是整个过程中唯一同步且不可中断的部分。React 会遍历副作用清单，一次性将所有的变更应用到真实的 DOM 节点上，并同步执行生命周期方法（如 componentDidUpdate）或 useEffect 的清理与执行逻辑。之所以将 setState 设计为这种异步批处理（Batching）模式，是为了避免短时间内频繁操作 DOM 导致的性能剧震，确保多次状态修改在一次重新渲染中完成，从而实现性能的最优化。

## 什么是Automatic Batching？

批处理（Batching） 是 React 为了减少不必要的重新渲染（Re-render）而采用的一种性能优化手段。在 React 18 之前，批处理主要局限于 React 原生事件处理函数内部，而在 setTimeout、Promise 或原生 DOM 事件中，setState 往往会触发同步更新。从 React 18 开始，框架引入了 自动批处理（Automatic Batching） 机制，无论状态更新发生在何处，React 都会自动将同一事件循环中的多次状态变更合并为一次渲染。这意味着，即便你连续调用了三次 setState，React 也只会进行一次虚拟 DOM 的对比和真实的 DOM 渲染，极大地提升了复杂应用在处理异步逻辑时的性能表现。

## React 的渲染与更新流程是什么？

React 的渲染与更新可以总结为一个由 JSX 转换、虚拟 DOM 构建及 Patch 补丁算法 共同驱动的闭环过程。

JSX 的本质是 React.createElement 函数的语法糖。在编译阶段，Babel 会将 JSX 转换为这种嵌套的函数调用；在执行阶段，这些函数运行并返回一个普通的 JavaScript 对象，即 虚拟 DOM（vnode）。这个对象描述了 DOM 节点的标签、属性和子节点。初次渲染时，React 会通过渲染器执行 patch(container, vnode)，根据虚拟 DOM 树完整地创建真实 DOM 节点，并将其挂载到页面的容器中。

当组件内部调用 setState 时，更新流程正式开启。React 并不会立即操作 DOM，而是将该组件标记为 “脏组件”（dirtyComponents） 并推入更新队列。随后，React 重新执行该组件的 render 方法（或函数组件体），结合最新的 props 和 state 生成一套全新的虚拟 DOM 树（newVnode）。此时，React 核心的 Diff 算法 开始介入，通过 patch(vnode, newVnode) 对新旧两棵树进行深度对比，找出最小差异。

在 Patch 过程中，React 遵循“同层比较”的策略，识别出哪些节点需要新增、移动或删除，并将这些变更收集到更新任务中。最终，React 进入提交阶段，将这些经过计算的最小化差异批量同步到真实 DOM 上。这种从“状态变更”到“虚拟 DOM 对比”再到“局部物理更新”的机制，不仅屏蔽了直接操作 DOM 的复杂性，也通过减少昂贵的 DOM 操作保证了复杂交互下的应用性能。

## 什么是diff算法？

React 的 Diff 算法是其调和（Reconciliation）机制的核心，它通过将传统 $O(n^3)$ 的树差异比较算法优化至 $O(n)$ 线性复杂度，实现了高效的视图更新。为了达到这种性能飞跃，React 基于 Web UI 的特点提出了三个前提假设：DOM 节点跨层级移动极少、相同类的组件产生相似结构、同级节点可以通过唯一 ID（key）区分。基于这些假设，Diff 过程被划分为三个层级的比较策略：Tree Diff（树比对）、Component Diff（组件比对）和 Element Diff（元素比对）。

- 在 Tree Diff 中，React 采用分层求异的策略，仅对同一层级的节点进行比较。如果一个节点在更新中跨越了层级（例如从 div 移到了 p 下），React 不会尝试移动它，而是直接销毁旧节点及其子树，并在新位置重新创建。
- 在 Component Diff 阶段，React 检查组件类型：如果类型不同，则判定结构完全改变，直接替换整个组件；如果类型相同，则保留实例，仅更新 props 并触发子节点的递归对比。此外，开发者可以通过 shouldComponentUpdate 或 PureComponent 手动跳过不必要的子树 Diff，这是性能优化的重要手段。
- Element Diff 是处理列表渲染时的关键。对于同一层级的多个子节点，React 默认按顺序比对，如果只是在头部插入一个元素，会导致后续所有节点被重新渲染。为了解决这个问题，React 引入了 key 属性。通过 key，React 可以快速识别出哪些节点是稳定的，从而实现节点的复用、移动或删除，而不是机械地销毁重建。在实现上，React 会经历两轮遍历：第一轮优先处理“更新”的节点，第二轮处理剩下的新增、删除或移动逻辑。这种“更新优先”的策略极大提升了高频交互场景下的响应速度

自 React 16 引入 Fiber 架构 后，Diff 算法的执行变得更加灵活。传统的 Diff 是递归执行的，一旦开始无法中断，容易造成页面卡顿；而 Fiber 将 Diff 任务拆分为微小的单元，使其具备了时间切片（Time Slicing）能力。React 可以在每一帧的空闲时间内进行虚拟 DOM 的对比，如果此时有更高优先级的任务（如用户输入），则暂停 Diff，等主线程空闲后再恢复。这种可中断、可恢复的特性，配合双缓存技术（current tree 与 workInProgress tree），使得 React 在处理超大型组件树时，依然能保持丝滑的用户体验。

## 为什么diff算法性能比传统比较好？

传统的树位移算法之所以达到 $O(n^3)$，是因为它试图在两棵树之间寻找“绝对最优”的转换方案。给定一棵拥有 $n$ 个节点的树，要找到将它转换为另一棵树的最少操作，通用的动态规划算法需要比较每一对节点之间的关系。这意味着首先要进行 $n \times n$ 次比较来确定节点间的对应关系，随后还需要处理节点在树中位置的变动、属性的更新以及复杂的递归逻辑，最终导致计算量级飙升至 $n$ 的三次方。对于一个拥有 1000 个节点的现代网页，这涉及到十亿次级别的计算，在浏览器端几乎会导致直接崩溃。

React 能够将复杂度降至 $O(n)$，核心在于它通过人为设定的限制条件，将一个“开放性命题”转化为了“填空题”。React 的 Diff 算法不再追求数学意义上的全球最优解，而是基于 Web UI 的实际应用场景采取了贪心策略。它假设开发者很少会将一个节点从树的顶端移动到最末端，因此它只进行同层比较（Level-by-level）。这意味着 React 只需要对整棵树进行一次深度优先遍历（DFS），每个节点只会被访问一次。

这种线性复杂度的实现依赖于三个具体的扫描机制：首先是类型匹配，如果节点类型变了（如 `div` 变 `p`），React 认为没必要再往下比了，直接整块替换；其次是组件边界，同类组件产生相似结构，不同类则不复用；最后是Key 值的辅助。在处理同级长列表时，如果没有 `key`，React 必须按索引一个个对比，一旦中间插一个元素，后面全得重绘。有了 `key` 之后，React 就像有了索引表，可以通过哈希映射（Hash Map）在 $O(1)$ 时间内找到旧节点并复用，从而确保整个 Diff 过程的时间消耗与节点总数 $n$ 成正比。

## React有哪些优化性能的手段？

在 React 函数组件中，`React.memo` 是最常用的性能优化高阶组件，其作用类似于类组件中的 `PureComponent`。它通过对组件接受的 `props` 进行浅比较（Shallow Comparison），来决定是否触发重新渲染。如果传入的 `props` 并没有发生变化，React 就会直接复用上一次的渲染结果，从而跳过复杂的 Diff 过程，这在父组件频繁更新但子组件状态稳定的场景下尤为有效。

```javascript
// 使用 memo 包裹子组件
const Child = React.memo(({ name }) => {
  console.log("子组件渲染了"); // 只有当 name 改变时才会打印
  return <div>你好，{name}</div>;
});

function Parent() {
  const [count, setCount] = useState(0);
  // 改变 count 会导致 Parent 重新执行，但 Child 的 props 没变
  return (
    <>
      <Child name="张三" />
      <button onClick={() => setCount(count + 1)}>点我次数: {count}</button>
    </>
  );
}
```

`useMemo` 提供了更细粒度的逻辑控制，它主要用于缓存复杂的计算结果。与 `React.memo` 控制整个组件的渲染不同，`useMemo` 允许你包裹一段耗时的计算逻辑，并定义依赖项数组。只有当依赖项发生变化时，才会重新执行该逻辑并返回新值，否则将始终返回缓存的旧值，这有效避免了在每次 `render` 过程中进行重复的高能耗计算。

```javascript
function CalculationDemo({ list }) {
  const [color, setColor] = useState("red");

  // 使用 useMemo 缓存计算密集型任务
  const expensiveValue = useMemo(() => {
    console.log("正在进行复杂计算...");
    return list.reduce((acc, cur) => acc + cur, 0);
  }, [list]); // 仅当 list 变化时重新计算

  return (
    <div style={{ color }}>
      结果: {expensiveValue}
      <button onClick={() => setColor(color === "red" ? "blue" : "red")}>
        切换颜色
      </button>
    </div>
  );
}
```

`useCallback` 则是专门用于缓存函数引用的 Hook。在 JavaScript 中，函数作为对象，每次组件重新渲染都会创建一个新的引用，这会导致即便使用了 `React.memo` 的子组件也会因为检测到“函数引用变了”而误触发重渲染。通过 `useCallback` 包裹回调函数，可以保证在依赖项不变的情况下，函数引用保持恒定，从而配合 `React.memo` 真正实现性能的闭环优化。

```javascript
const MemoButton = React.memo(({ onClick }) => {
  console.log("按钮组件渲染了");
  return <button onClick={onClick}>提交</button>;
});

function Parent() {
  const [text, setText] = useState("");

  // 使用 useCallback 确保函数引用在重新渲染时不改变
  const handleClick = useCallback(() => {
    console.log("提交内容:", text);
  }, [text]); // 只有当 text 变化时，函数引用才会更新

  return (
    <>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <MemoButton onClick={handleClick} />
    </>
  );
}
```

在处理列表渲染时，使用唯一 ID 作为 `key` 是最基础且关键的优化手段。如果使用数组下标 `index` 作为 `key`，当列表发生插入、删除或排序操作时，会导致 React 无法正确识别节点的稳定性，进而引发大量的 DOM 销毁与重建。而唯一的 ID 能让 React 的 Diff 算法实现 $O(1)$ 级别的节点寻址，确保只有发生变化的元素才会更新。

对于频繁切换显示状态的组件，通过 CSS 样式（如 `display: none`）隐藏 往往比通过条件判断完全销毁节点性能更好。虽然条件渲染能减少 DOM 节点的基数，但频繁的节点创建与挂载会带来昂贵的布局计算开销。如果组件体积较大且切换频繁，预先加载并利用 CSS 隐藏可以实现更平滑的视觉反馈。

利用 `Suspense` 和 `React.lazy` 进行代码分割（Code Splitting） 是优化首屏加载速度的核心。通过懒加载，React 只会在组件真正需要被渲染时才去加载对应的二进制脚本文件。这不仅减少了首屏 Bundle 的体积，还能配合 `Suspense` 优雅地展示加载占位图（Loading Spinner），显著提升了大中型应用的感知性能和用户留存。

## 类组件和函数组件的区别是什么？

类组件的核心在于生命周期（Lifecycle）。由于它是一个类实例，React 会在组件的整个生命周期中保持这个实例，开发者需要手动在不同的阶段（如挂载、更新、卸载）处理副作用。这种模式虽然结构清晰，但往往会导致逻辑碎片化——例如，订阅和取消订阅的代码必须被迫拆分在两个不同的生命周期方法中，增加了维护难度。

相比之下，函数组件通过 Hooks 彻底改变了逻辑组织方式。`useEffect` 允许你将相关的逻辑（如订阅与清理）放在同一个代码块内，使代码更加内聚。最关键的区别在于：函数组件捕获了渲染时的值（Capture Value）。由于闭包特性，函数组件在每次渲染时都有其独立的作用域，这避免了类组件中因 `this` 指向变化导致的异步回调数据不一致问题，使得 UI 与状态的关系更加稳定、可预测。

| 特性     | 类组件 (Class Component)                         | 函数组件 (Function Component)            |
| -------- | ------------------------------------------------ | ---------------------------------------- |
| 编程范式 | 面向对象 (OOP)，依赖 `this` 引用                 | 函数式编程 (FP)，纯函数思想              |
| 状态管理 | 使用 `this.state` 和 `this.setState`             | 使用 `useState` 等 Hooks                 |
| 生命周期 | 拥有完整的生命周期钩子（如 `componentDidMount`） | 使用 `useEffect` 模拟生命周期行为        |
| 逻辑复用 | 主要依靠 HOC (高阶组件) 或 Render Props          | 通过 Custom Hooks (自定义 Hook) 轻松复用 |
| 性能优化 | `shouldComponentUpdate` 或 `PureComponent`       | `React.memo`、`useMemo` 和 `useCallback` |
| 代码体积 | 编译后的代码较多，`this` 指向易混淆              | 代码更简洁，Tree-shaking 友好，体积更小  |
| 未来趋势 | 官方保留支持，但不再是开发首选                   | React 官方推荐，社区生态的主流方案       |

## useLayoutEffect 和 useEffect 的区别是什么？

useEffect 和 useLayoutEffect 的核心区别在于执行时机以及对浏览器渲染阻塞的影响。useEffect 是在浏览器完成布局（Layout）和涂色（Paint）之后异步执行的，这意味着它不会阻塞页面的渲染过程。这种设计旨在保证应用的流畅性，适用于大多数副作用场景，如数据获取、订阅事件或手动修改不影响首屏视觉的 DOM。由于它是异步的，如果副作用涉及直接修改 DOM 样式，用户可能会看到页面先以旧状态渲染，随后瞬间跳转到新状态的“闪烁”现象。

相比之下，useLayoutEffect 会在 DOM 更新之后、浏览器绘制屏幕之前同步触发。它会阻塞浏览器的渲染链路，直到副作用逻辑执行完毕。这种特性使其成为处理 UI 稳定性 问题的利器：当你需要测量 DOM 元素的尺寸、位置，并根据这些信息立即调整样式时，useLayoutEffect 能确保这些变更与原始更新在同一个渲染帧内完成，从而彻底消除视觉上的闪烁。然而，由于其同步阻塞的特性，应谨慎处理内部逻辑，避免因计算过重导致页面掉帧或交互卡顿。

## React中如何使用Axios获取数据？

在 React 中，使用 Axios 获取数据的标准做法是在 `useEffect` Hook 中发起异步请求。由于数据获取属于副作用（Side Effect），我们需要确保它在组件挂载后执行。通常，我们会定义一个 `state` 来存储获取到的数据，并使用 `try...catch` 块来处理请求过程中可能出现的错误。此外，为了提升用户体验，通常还会维护一个 `loading` 状态，以便在数据加载期间展示 loading 效果或占位符。

在编写请求逻辑时，处理异步函数的闭包问题和组件卸载时的清理至关重要。你应当在 `useEffect` 内部定义一个 `async` 函数并调用它，而不是直接将 `useEffect` 的回调函数设为 `async`。为了防止组件卸载后异步请求依然尝试更新状态（导致内存泄漏警告），建议使用 `AbortController` 来取消尚未完成的请求。通过将请求逻辑封装在自定义 Hook（如 `useFetch`）中，不仅可以提高代码的复用性，还能让业务组件的逻辑更加纯粹、易于测试。

## 受控组件和非受控组件？

在 React 中，受控组件与非受控组件的核心区别在于“数据控制权的归属”。受控组件将表单状态交由 React 的 state 统一管理，通过 value 绑定和 onChange 回调实现数据的双向流向，其优势在于能实时拦截输入、进行格式校验和逻辑联动，体现了“单一数据源”的原则。而非受控组件则保留了原生 DOM 的特性，状态由浏览器自身维护，开发者通过 useRef 在需要时手动从 DOM 节点中读取数据，这种方式代码量更少，更适合处理逻辑简单的表单或集成非 React 的第三方库（如富文本编辑器）。

## 组件之间如何通讯？

父子组件通信：父组件通过 Props 将数据、状态或函数直接传递给子组件；而子组件若想影响父组件，则通过调用父组件传递过来的回调函数（Callback）来实现。

Context API：React 原生的“依赖注入”机制，它允许你在父树顶端定义数据，让下层所有组件都能直接消费，有效解决了深层嵌套的通信问题，常用于主题、语言包或用户信息。

Redux： 提供了一个中心化的 Store 和严格的更新流（Action -> Reducer -> Store），适用于逻辑极其复杂、状态多处共享且需要“时间旅行”调试的大型应用。

## 什么是useEffect的闭包陷阱？

函数组件的每一次渲染都有其独立的作用域、变量和副作用。

场景一：闭包陷阱（错误示例）。在这个例子中，你会看到由于依赖项缺失，导致定时器“永远活在过去”的现象。

```javascript
function ClosureTrap() {
  const [value, setValue] = useState(0);

  useEffect(() => {
    // 这里的匿名函数是在组件【初次挂载】时创建的
    // 它形成了一个闭包，捕获了当时作用域下的 value (值为 0)
    const timer = setInterval(() => {
      // 无论外部的 value 如何变化，这个闭包由于没有被重新触发
      // 它引用的永远是第一次渲染时的变量空间中的 0
      console.log("定时器读取的值:", value);
    }, 1000);

    // 依赖数组为空，意味着该 Effect 永不重新执行，timer 永不销毁
  }, []);

  return (
    <div>
      <p>当前值: {value}</p>
      <button onClick={() => setValue(value + 1)}>累加</button>
    </div>
  );
}
```

场景二：正确更新（标准解法）。通过同步依赖项和清理逻辑，我们确保了每次状态变化都能产生一个新的、引用正确变量的闭包。

```javascript
function CorrectUpdate() {
  const [value, setValue] = useState(0);

  useEffect(() => {
    // 每次 value 变化，React 都会执行这个副作用函数
    // 此时的 value 是当前这次渲染快照中的最新值
    const timer = setInterval(() => {
      console.log("定时器读取的值:", value);
    }, 1000);

    // 【关键点】：返回清理函数
    // 下一次 Effect 执行前，React 会先执行这个函数销毁旧定时器
    // 从而防止多个定时器重叠执行以及内存泄漏
    return () => {
      clearInterval(timer);
      console.log("旧定时器已清除，即将同步新值...");
    };

    // 将 value 加入依赖，确保 state 变化时重新运行副作用
  }, [value]);

  return (
    <div>
      <p>当前值: {value}</p>
      <button onClick={() => setValue(value + 1)}>累加</button>
    </div>
  );
}
```

## Suspense 与 Lazy 如何使用？

在 React 中，Suspense 是一种声明式的组件加载机制，它允许你“等待”某些异步操作（如组件代码加载或数据获取）完成，并在等待期间自动展示一个占位 UI（Fallback）。它的核心逻辑是：当子组件树中某个组件尚未准备好渲染时，它会向上抛出一个“异常”（通常是一个 Promise），Suspense 捕获到这个状态后，会暂时隐藏未就绪的子组件，转而渲染 fallback 属性定义的加载内容。

目前 Suspense 最经典的应用场景是配合 React.lazy 实现代码分割（Code Splitting）。在大型应用中，我们不希望一次性加载所有页面的 JS 代码，通过 lazy 包装组件，配合 Suspense 容器，可以实现“用到才加载”的效果。当用户跳转到某个路由时，浏览器才开始下载该组件的资源，此时 Suspense 会展示 Loading 动画，避免了页面出现空白或逻辑断层，极大地优化了首屏加载速度。

```javascript
import React, { Suspense, lazy } from "react";

// 1. 使用 lazy 异步导入组件
const HeavyComponent = lazy(() => import("./HeavyComponent"));

function App() {
  return (
    <div>
      <h1>我的应用</h1>
      {/* 2. 使用 Suspense 包裹异步组件，并提供 fallback */}
      <Suspense fallback={<div>正在努力加载中...</div>}>
        <HeavyComponent />
      </Suspense>
    </div>
  );
}
```
