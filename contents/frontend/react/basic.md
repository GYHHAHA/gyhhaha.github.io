---
short_title: React Core
---

# React 基础

## 官方文档

### 核心概念

React 组件是常规的 JavaScript 函数，但它们的名字总是以大写字母开头，且返回 JSX 标签。如果你的标签和 return 关键字不在同一行，则必须把它包裹在一对括号中，否则任何在 return 下一行的代码都将被忽略。导入组件的时候，可以不写 `.js` 如 `import A from './A'`。

JSX 只能返回一个根元素，里面有多个子元素的时候，可以使用 `<>...</>` 或者 `React.Fragment` （使用 key 时）代替。由于 class 是一个保留字，所以在 JSX 标签中需要用 className 来代替。所有可用属性的列表见 [官方文档](https://zh-hans.react.dev/reference/react-dom/components/common#common-props)。JSX 可以使用 `{}` 来使用 JavaScript 中的变量和表达式，大括号只能用在用作 JSX 标签内的文本和用作紧跟在 = 符号后的属性这两种情况。当你需要内联样式的时候，你可以给style属性传递一个对象。

```javascript
import React from "react";

function Example() {
  const name = "小明";
  const isVip = true;
  const age = 18;

  return (
    <>
      <h1 className="title">你好，{name}</h1>

      <p>年龄：{age}</p>

      {isVip && (
        <span style={{ color: "red", fontWeight: "bold" }}>VIP 用户</span>
      )}
    </>
  );
}

export default Example;
```

props 是组件的唯一参数，React 组件函数接受一个参数，当不需要所有值时，使用解构。在没有指定值的情况下给 prop 一个默认值，可通过在参数后面写 = 和默认值来进行解构将内容嵌套在 JSX 标签中时，父组件将在名为 children 的 props 中接收到该内容。props 是不可变的，当一个组件需要改变它的 props 时，它不得不请求它的父组件传递不同的 props。

```javascript
// 子组件
function Card({ title = "默认标题", count = 0, children }) {
  return (
    <div style={{ border: "1px solid #ccc", padding: "10px", margin: "10px" }}>
      <h2>{title}</h2>
      <p>数量：{count}</p>
      <div>{children}</div>
    </div>
  );
}

// 父组件
function App() {
  return (
    <div>
      {/* 传递所有 props */}
      <Card title="商品信息" count={5}>
        <p>这是通过 children 传递的内容</p>
      </Card>

      {/* 不传 title 和 count，会使用默认值 */}
      <Card>
        <button>点击按钮</button>
      </Card>
    </div>
  );
}
```

可以选择性地将一些 JSX 赋值给变量，然后用大括号将其嵌入到其他 JSX 中。`{cond ? <A /> : <B />}` 表示当cond为真值时，渲染 `<A />`，否则 `<B />`。`{cond && <A />}` 表示当cond为真值时，渲染`<A />`，否则不进行渲染。

```javascript
type Props = {
  isLogin: boolean;
  isAdmin: boolean;
};

function UserPanel({ isLogin, isAdmin }: Props) {
  let statusMessage;
  if (isLogin) {
    statusMessage = <p>欢迎回来！</p>;
  } else {
    statusMessage = <p>请先登录</p>;
  }
  return (
    <div>
      <h2>用户中心</h2>
      {/* 方式一：变量 */}
      {statusMessage}
      {/* 方式二：三元表达式 */}
      <p>
        当前身份：
        {isLogin ? "已登录" : "未登录"}
      </p>
      {/* 方式三：&& 条件渲染 */}
      {isAdmin && <button>进入后台管理</button>}
    </div>
  );
}
```

key 用来帮助 React 识别哪些元素发生了变化，从而进行高效、正确的更新。列表的渲染一般通过 `map` 将数组对象映射到 JSX 对象，并通过大括号嵌入组件返回值中。请记住 key 不是全局唯一的。它们只能指定 父组件内部 的顺序。

```javascript
function List() {
  const items = [
    { id: 1, name: "苹果" },
    { id: 2, name: "香蕉" },
    { id: 3, name: "橙子" },
  ];

  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

### 添加交互

在 React 中，大多数事件都会向上“冒泡”，也就是说当你点击按钮时，会先执行按钮的 onClick，然后再执行父元素的 onClick。如果在子元素的事件处理函数中调用 e.stopPropagation()，就可以阻止事件继续向上传播。需要注意的是，onScroll 是 React 中唯一不会冒泡的事件，它只会在绑定它的那个元素上触发，不会传递给父元素。

```javascript
export default function App() {
  return (
    <div
      onClick={() => alert("点击了父元素")}
      style={{ border: "1px solid black", padding: "20px" }}
    >
      <button
        onClick={(e) => {
          e.stopPropagation(); // 阻止冒泡
          alert("点击了按钮");
        }}
      >
        点我
      </button>

      <div
        onScroll={() => alert("触发了滚动")}
        style={{ height: "50px", overflow: "auto", marginTop: "10px" }}
      >
        <div style={{ height: "150px" }}>滚动我</div>
      </div>
    </div>
  );
}
```

在 React 中，事件分三个阶段传播：捕获阶段 → 目标阶段 → 冒泡阶段。当点击按钮时，会先从外到内执行 onClickCapture（捕获阶段），然后执行被点击元素本身的 onClick（目标阶段），最后从内到外执行 onClick（冒泡阶段）。如果在子元素中调用 e.stopPropagation()，会阻止冒泡阶段继续向上传播，但不会影响已经发生的捕获阶段。因此上面例子中点击按钮时的执行顺序是：先弹出“父元素 capture”，再弹出“按钮 click”，而“父元素 bubble”不会执行。捕获阶段常用于埋点、路由监听等需要在最外层优先拦截事件的场景。

```javascript
export default function App() {
  return (
    <div
      onClickCapture={() => alert("父元素 capture")}
      onClick={() => alert("父元素 bubble")}
      style={{ border: "1px solid black", padding: "20px" }}
    >
      <button
        onClick={(e) => {
          e.stopPropagation();
          alert("按钮 click");
        }}
      >
        点我
      </button>
    </div>
  );
}
```

```{tip} 表单事件
表单按钮会触发表单提交事件，默认将重新加载整个页面，调用e.preventDefault()来阻止。
```

当一个组件需要在不同渲染之间保存某些数据（比如当前索引、输入值、计数等），就需要使用状态（state）。在函数组件中通过 useState Hook 来声明 state 变量，它返回当前 state 和更新它的 setter 函数，调用 setter 会触发组件重新渲染。useState 和其他 Hook 必须在组件或自定义 Hook 的顶层调用，不能放在条件或循环中。每个 state 变量都是私有且隔离的，同一个组件渲染多次时每个实例都有独立的 state。有两种原因会导致组件的渲染：组件的初次渲染，组件（或者其祖先之一）的状态发生了改变。

下面这段代码，之所以 3 秒后打印的是 0，是因为 setTimeout 里的回调函数捕获了点击那一刻那次渲染的 state 快照。在 React 中，每次渲染都会生成一份独立的 state 值，事件处理函数会“记住”它创建时所在那次渲染的变量。当你点击按钮时，当前 number 是 0，setNumber(number + 5) 只是触发下一次渲染，并不会改变当前闭包里的 number。因此 3 秒后执行 alert(number) 时，读到的仍然是那次点击时保存的 0，而不是更新后的 5。这种现象叫做“过期闭包（stale closure）”。

```javascript
import { useState } from "react";

export default function Counter() {
  const [number, setNumber] = useState(0);

  return (
    <>
      <h1>{number}</h1>
      <button
        onClick={() => {
          setNumber(number + 5);
          setTimeout(() => {
            alert(number);
          }, 3000);
        }}
      >
        +5
      </button>
    </>
  );
}
```

如果想在下次渲染之前多次更新同一个state，可以使用更新函数，第一个例子只会加一，此时setState(x)实际上会像setState(n => x)一样运行，第二个例子会加三，因为在下次渲染期间调用useState时，React会遍历队列，获取上一个更新函数的返回值，并将其作为参数传递给下一个更新函数，以此类推：

```javascript
import { useState } from "react";

export default function Counter() {
  const [number, setNumber] = useState(0);

  return (
    <>
      <h1>{number}</h1>
      <button
        onClick={() => {
          setNumber((n) => n + 1);
          setNumber((n) => n + 1);
          setNumber((n) => n + 1);
        }}
      >
        +3
      </button>
    </>
  );
}
```

数组是另外一种可以存储在 state 中的 JavaScript 对象，它虽然是可变的，但是却应该被视为不可变。同对象一样，当你想要更新存储于 state 中的数组时，你需要创建一个新的数组（或者创建一份已有数组的拷贝值），并使用新数组设置 state。

| 操作类型 | 避免使用（会改变原始数组）    | 推荐使用（会返回一个新数组）                      |
| -------- | ----------------------------- | ------------------------------------------------- |
| 添加元素 | `push`，`unshift`             | `concat`，`[...arr]` 展开语法                     |
| 删除元素 | `pop`，`shift`，`splice`      | `filter`，`slice`                                 |
| 替换元素 | `splice`，`arr[i] = ...` 赋值 | `map`                                             |
| 排序     | `reverse`，`sort`             | 先将数组复制一份再排序（例如：`[...arr].sort()`） |

### 状态管理

只要一个组件还被渲染在 UI 树的相同位置，React 就会保留它的 state。 如果它被移除，或者一个不同的组件被渲染在相同的位置，那么 React 就会丢掉它的 state。有时候，你可能想要重置一个组件的 state，一般有两种方法：将组件渲染在不同的位置、使用 key 赋予每个组件一个明确的身份。

当组件里有多个状态变量互相影响、并且逻辑复杂时直接写很多 state 和对应 setter 会难以维护，这时候可以用 useReducer 统一管理逻辑，其结构如下：

```javascript
// state 是当前状态对象
// action.type 表示意图（加、减、更新字段等）
// reducer 负责根据 action 生成新状态
function reducer(state, action) {
  switch (action.type) {
    case "field":
      return {
        ...state,
        [action.fieldName]: action.payload,
      };
    case "reset":
      return initialState;
    default:
      return state;
  }
}
```

在组件中：

```javascript
const [state, dispatch] = useReducer(reducer, initialState);

// 调用 dispatch 触发状态变化
dispatch({
  // 意图
  type: "field",
  // 字段
  fieldName: "firstName",
  payload: e.target.value,
});
```

在 React 中默认数据是从上到下单向传递，每一层组件都要传 props，而Context 允许在组件树中跨层级传递数据，而无需逐层 props 传递。

1. 创建 Context

```javascript
// defaultValue 是在没有 Provider 时的默认值（可选）
const MyContext = React.createContext(defaultValue);
```

2. 提供（Provider）Context

```javascript
<MyContext.Provider value={someValue}>
  <App />
</MyContext.Provider>
```

3. 读取 Context

```javascript
const value = useContext(MyContext);
```

下面的例子中，Provider 把 "dark" 放进 Context，不管 ThemedButton 是多深层级，它都能通过 useContext 直接读到 "dark"。React 会在组件树上向下查找最近的 Provider，然后把它的 value 提供给后代消费组件。

```javascript
const ThemeContext = React.createContext("light");

function Toolbar() {
  return (
    <ThemeContext.Provider value="dark">
      <ThemedButton />
    </ThemeContext.Provider>
  );
}

function ThemedButton() {
  const theme = useContext(ThemeContext);
  return <button className={theme}>按钮</button>;
}
```

Context 的 value 必须保持引用稳定，否则会导致所有消费者组件重新渲染。下面这个例子中，当点击按钮后，App 重新 render，value={{ theme: "dark" }} 重新创建，新对象 !== 旧对象，React 认为 Context 变了，所有 useContext 的组件全部重新渲染，即使 theme 没变。

```javascript
function App() {
  const [count, setCount] = useState(0);

  return (
    <MyContext.Provider value={{ theme: "dark" }}>
      <Child />
      <button onClick={() => setCount(count + 1)}>+</button>
    </MyContext.Provider>
  );
}
```

正确做法是用 useMemo 保持引用稳定：

```javascript
const value = useMemo(() => {
  return { theme: "dark" };
}, []);

<MyContext.Provider value={value}>
```

### 脱围机制

当你希望组件“记住”某些信息，但又不想让这些信息触发新的渲染时，可以使用 ref。你可以用 ref.current 属性访问该 ref 的当前值。这个值是有意被设置为可变的，意味着你既可以读取它也可以写入它。ref 用来存储 timeout ID、DOM 元素、不需要被用来计算 JSX 的其他对象，此外还可以像其它 prop 一样将 ref 从父组件传递给子组件。

```javascript
import React, { useRef } from "react";

function FocusInput() {
  // 创建一个ref对象来存储对输入元素的引用
  const inputRef = useRef(null);

  // 一个函数，当按钮被点击时调用，用来设置输入元素的焦点
  const focusOnInput = () => {
    // 使用current属性访问ref引用的DOM节点
    if (inputRef.current) {
      inputRef.current.focus();
    }
  };

  return (
    <div>
      <input ref={inputRef} type="text" placeholder="点击按钮来聚焦我" />
      <button onClick={focusOnInput}>聚焦输入框</button>
    </div>
  );
}

export default FocusInput;
```

useEffect 在 DOM 更新（commit）之后执行，对于下面的例子，函数在初次渲染后执行、deps 改变后再执行、cleanup 在下次执行前或卸载时执行。当你有订阅、计时器、监听器等副作用时，需要通过 cleanup 清理它们避免资源泄漏。第 1 次渲染完成后执行所有 useEffect，按组件树深度从上到下执行 cleanup 和副作用，下一次渲染 deps 改变时执行 cleanup 再执行新 effect。

```javascript
useEffect(() => {
  // 依赖变化时执行
  return () => {
    // 卸载时执行
  };
}, [dep1, dep2]);
```

以数据库的例子，你可能希望在组件挂载后连接数据库，在组件卸载后断开连接。

```javascript
useEffect(() => {
  const connection = createConnection();
  connection.connect();
  return () => {
    connection.disconnect();
  };
}, []);
```

依赖数组可以包含多个依赖项，当指定的所有依赖项在上一次渲染期间的值与当前值完全相同时，React会跳过重新运行该Effect：

```javascript
useEffect(() => {
  // 这里的代码会在每次渲染后执行
});

useEffect(() => {
  // 这里的代码只会在组件挂载后执行
}, []);

useEffect(() => {
  //这里的代码只会在每次渲染后，并且 a 或 b 的值与上次渲染不一致时执行
}, [a, b]);
```

useLayoutEffect 在浏览器绘制之前执行，一般用在需要读 DOM 尺寸或计算布局，或者需要在 React 绘制之前修改 DOM 的时候，注意这个函数会阻塞浏览器绘制。。

```javascript
import { useState, useRef, useLayoutEffect } from "react";

export default function Example() {
  const [height, setHeight] = useState(0);
  const divRef = useRef(null);

  useLayoutEffect(() => {
    const rect = divRef.current.getBoundingClientRect();
    setHeight(rect.height);
  }, []);

  return (
    <div>
      <div ref={divRef} style={{ background: height > 50 ? "red" : "blue" }}>
        这是一段可能很长的文本内容
      </div>
    </div>
  );
}
```

当组件重新渲染的时候，如果不希望引起某些无关值重新计算，可以使用useMemo，它会在渲染期间执行：

```javascript
import { useMemo, useState } from "react";

function TodoList({ todos, filter }) {
  const [newTodo, setNewTodo] = useState("");
  // ✅ 除非 todos 或 filter 发生变化，否则不会重新执行 getFilteredTodos()
  const visibleTodos = useMemo(
    () => getFilteredTodos(todos, filter),
    [todos, filter],
  );
  // ...
}
```

使用useEffect获取数据的例子：

```javascript
function SearchResults({ query }) {
  const [page, setPage] = useState(1);
  const params = new URLSearchParams({ query, page });
  const results = useData(`/api/search?${params}`);

  function handleNextPageClick() {
    setPage(page + 1);
  }
  // ...
}

function useData(url) {
  const [data, setData] = useState(null);
  useEffect(() => {
    let ignore = false;
    fetch(url)
      .then((response) => response.json())
      .then((json) => {
        if (!ignore) {
          setData(json);
        }
      });
    return () => {
      ignore = true;
    };
  }, [url]);
  return data;
}
```

Effect 只用来“和外部世界同步”，只要能在渲染阶段算出来，就不要用 Effect。

| 目的                      | 正确做法               | 不要做什么            |
| ------------------------- | ---------------------- | --------------------- |
| 根据 state/props 计算新值 | 直接在 render 里算     | 用 Effect 再 setState |
| 缓存昂贵计算              | useMemo                | Effect + state        |
| 重置整个组件 state        | 改 key                 | Effect 里 reset       |
| 部分 state 跟随 prop 改变 | render 中判断          | Effect 里 setState    |
| 事件触发逻辑（点击/提交） | 写在事件函数里         | 放进 Effect           |
| 多个 state 连锁更新       | 在事件里一次算完       | 写成 Effect 链        |
| 通知父组件                | 在事件中直接调用       | Effect 里调用         |
| 表单提交                  | 在 submit 里发请求     | 用 Effect 监听状态    |
| 订阅外部 store            | useSyncExternalStore   | 手写 Effect           |
| 数据获取                  | Effect（带清理）或框架 | 忽略竞态问题          |

组件的生命周期包括挂载、更新和卸载，而Effect只有开始同步和停止同步，而且这个循环可能发生很多次。

Effect读取的所有响应式值，都必须写进依赖，但Effect Event允许在Effect内部，读取最新state，但不触发重新同步。

```javascript
// 下面注释的代码，切换 theme 时会重新连接聊天室。
// useEffect(() => {
//   connection.on('connected', () => {
//     showNotification('Connected!', theme);
//   });
// }, [roomId, theme]);

const onConnected = useEffectEvent(() => {
  showNotification("Connected!", theme);
});

useEffect(() => {
  connection.on("connected", () => {
    onConnected();
  });
}, [roomId]);
```
