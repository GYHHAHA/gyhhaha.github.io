---
short_title: React Core
---

# React 基础

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

key 用来帮助 React 识别哪些元素发生了变化，从而进行高效、正确的更新。列表的渲染一般通过 `map` 将数组对象映射到 JSX 对象，并通过大括号嵌入组件返回值中。

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
