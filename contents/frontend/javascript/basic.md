---
short_title: ES6 Core
---

# JavaScript 基础

## 语言基础

基本数据类型
: JavaScript 一共有 7 种基本数据类型：number / string / boolean / null / undefined / symbol / bigint。它们的共同特点：存储在栈内存，直接存储值本身，不可变（immutable），按值比较，没有方法（方法来自包装对象）。

| 类型      | 说明     | 详细拆解                                                                |
| --------- | -------- | ----------------------------------------------------------------------- |
| number    | 数字类型 | 基于 IEEE 754 双精度浮点数；包含 NaN、Infinity；存在精度问题（0.1+0.2） |
| string    | 字符串   | 不可变；按 UTF-16 编码；拼接会生成新字符串                              |
| boolean   | 布尔值   | 只有 true / false；常用于逻辑判断和隐式转换                             |
| null      | 空值     | 表示“人为赋空”；typeof 为 "object"（历史遗留问题）                      |
| undefined | 未定义   | 变量声明未赋值；函数无返回值；访问不存在属性                            |
| symbol    | 唯一值   | ES6 新增；保证唯一；常用于对象属性防冲突                                |
| bigint    | 大整数   | ES2020 新增；表示超大整数；不能与 number 混合运算                       |

引用数据类型
: JavaScript 中引用类型本质上只有一种：Object。数组、函数、日期等都是 Object 的不同表现形式。特点：存储在堆内存，变量中存的是地址，可变（mutable），按引用比较。

| 类型      | 说明     | 详细拆解                                |
| --------- | -------- | --------------------------------------- |
| Object    | 普通对象 | 键值对结构；属性可动态添加              |
| Array     | 数组     | 特殊对象；带 length；索引本质是字符串键 |
| Function  | 函数     | 可调用对象；具有 prototype；也是对象    |
| Date      | 日期对象 | 处理时间；底层基于时间戳                |
| RegExp    | 正则对象 | 用于模式匹配；test / exec 方法          |
| Map / Set | ES6 集合 | 键可为任意类型；解决对象键限制问题      |

typeof 和 instanceof 的区别
: typeof 用于判断数据类型，适合基本类型（除了 null）和 function；instanceof 用于判断对象是否属于某个构造函数，通过原型链判断。

|             | typeof          | instanceof |
| ----------- | --------------- | ---------- |
| 判断依据    | 值的类型标签    | 原型链     |
| 适合        | 基本类型        | 引用类型   |
| null        | "object"（bug） | false      |
| 可跨 iframe | 是              | 可能失效   |

各种空的比较
: 用===时，只有NaN不等于自己，用==时见下表：

| ==        | null  | undefined | 0     | false | ""    | NaN   |
| --------- | ----- | --------- | ----- | ----- | ----- | ----- |
| null      | true  | true      | false | false | false | false |
| undefined | true  | true      | false | false | false | false |
| 0         | false | false     | true  | true  | true  | false |
| false     | false | false     | true  | true  | true  | false |
| ""        | false | false     | true  | true  | true  | false |
| NaN       | false | false     | false | false | false | false |

var / let / const 总结
: 见下列表：

- var：函数作用域，存在变量提升，可重复声明，没有块级作用域。

```javascript
if (true) {
  var a = 1;
}

console.log(a); // 1
```

- let：块级作用域，不可重复声明，存在暂时性死区（TDZ），即从作用域开始到变量声明之间的区域。

```javascript
if (true) {
  let b = 2;
}

console.log(b); // 报错
```

- const：块级作用域，必须初始化，值不可重新赋值（但引用类型可修改）。

`for (var/let/const i = 0; i < 3; i++) { setTimeout(() => console.log(i), 0) }`输出什么？
: 如果使用 var，会输出 3 3 3，因为 var 是函数作用域，整个循环只存在一个共享的 i，当定时器回调真正执行时，循环早已结束，此时 i 已变为 3，所以打印三次 3；如果使用 let，会输出 0 1 2，因为 let 是块级作用域，每次循环都会创建一个新的 i 绑定，回调函数各自闭包保存了当次循环的独立变量值；而如果使用 const，代码会直接报错，因为 for 循环中的 i++ 需要对变量重新赋值，而 const 变量不能被修改。

解构
: 把结构里的值按位置/键名取出来赋给变量。对象解构靠 key 名，数组解构靠顺序。解构默认值只在 解构结果为 undefined 时生效（为 null 不会触发默认值）。

```javascript
const arr = [10, 20, 30];
const [a, b, c = 99] = arr; // 默认值

const obj = { name: "A", age: 18 };
const { name, age: userAge, city = "BJ" } = obj; // 重命名 + 默认值

// 嵌套解构
const {
  info: { email },
} = { info: { email: "x@x.com" } };
```

可选链
: 安全访问深层属性/方法：遇到 null 或 undefined 立刻返回 undefined，不再继续取值，避免报错。注意，遇到其他假值（0、""、false）不会短路。

```javascript
user?.profile?.name;
user?.getName?.(); // 方法存在才调用
arr?.[0]; // 可选链访问下标
```

空值合并
: 只在左侧是 null 或 undefined 时才使用右侧默认值（不会把 0/""/false 当成“空”）。如果你希望“空字符串也走默认值”，才用 || 或自定义判断。

```javascript
const count = input ?? 0;
0 || 100; // 100  （把 0 当成空）
0 ?? 100; // 0    （更符合“缺省值”的语义）
```

增强for
: for...of 用于遍历可迭代对象的“值”，而 for...in 用于遍历对象的“键”（包括可枚举的原型链属性）。

| 语法     | 遍历内容    | 适用对象                              | 示例                        |
| -------- | ----------- | ------------------------------------- | --------------------------- |
| for...of | 值（value） | 可迭代对象（Array、String、Map、Set） | `for (let v of [1,2,3]) {}` |
| for...in | 键（key）   | 对象（Object）                        | `for (let k in obj) {}`     |

## 常见特性

- **执行上下文**、变量提升/函数提升
- **作用域链**、自由变量
- **闭包** ⭐：常见题（循环+定时器、私有变量、缓存、模块化）
- **垃圾回收 & 内存泄漏常见坑**：闭包引用、DOM 引用未释放、定时器/事件未解绑
- **this 绑定规则** ⭐：默认/隐式/显式（call/apply/bind）/new、箭头函数 this
- **原型与原型链** ⭐：`prototype`、`__proto__`、`constructor`
- **继承方式**：原型链继承、构造函数继承、组合继承、寄生组合、ES6 `class extends/super`
- **new 做了什么** ⭐：手写 `new`
- 常见手写：`call/apply/bind`、`instanceof`
- **函数参数**：默认参数、剩余参数、arguments、柯里化
- **高阶函数**：map/reduce/filter
- **防抖/节流** ⭐（必会手写 + 适用场景）`debounce`、`throttle`
- **深拷贝/浅拷贝** ⭐：结构化克隆、JSON 限制、循环引用处理 `deepClone`（含循环引用）
- **手写系列**：`compose/pipe`、`once`、`memoize`

Proxy
: Proxy 用来创建一个对象的“代理”，可以拦截并自定义对象的基本操作（读、写、删、函数调用等）。`const proxy = new Proxy(target, handler)`中，`target` 是被代理的对象，`handler` 是一个对象，用于定义拦截操作的行为。常见的可以拦截操作包括：

| trap           | 作用        |
| -------------- | ----------- |
| get            | 读取属性    |
| set            | 设置属性    |
| deleteProperty | 删除属性    |
| has            | in 操作符   |
| ownKeys        | Object.keys |
| apply          | 函数调用    |
| construct      | new 调用    |

Reflect
: Reflect 是一个内置对象，提供和 Proxy 拦截操作对应的“默认行为”。

```javascript
// vue的响应式
const reactive = new Proxy(obj, {
  get(target, key) {
    track(target, key); // 依赖收集
    return Reflect.get(target, key);
  },
  set(target, key, value) {
    const result = Reflect.set(target, key, value);
    trigger(target, key); // 派发更新
    return result;
  },
});
```

```javascript
// 校验
const user = new Proxy(
  {},
  {
    set(target, key, value) {
      if (key === "age" && typeof value !== "number") {
        throw new Error("age 必须是数字");
      }
      return Reflect.set(target, key, value);
    },
  },
);
```

## 异步机制

Event Loop
: JavaScript 是单线程的，通过「调用栈 + 任务队列」机制实现异步，这个调度过程就叫事件循环。JS 是单线程，同一时间只能执行一个任务，但要处理异步（定时器、网络请求、DOM 事件），于是引擎把任务分成同步任务和异步任务。异步任务执行完成后进入任务队列，等待主线程空闲再执行。

宏任务和微任务
: 宏任务（MacroTask）是事件循环中的“大任务”，每一轮事件循环都会从宏任务队列中取出一个任务执行，常见的宏任务包括整体脚本（script）、setTimeout、setInterval、setImmediate（Node 环境）、I/O 操作以及浏览器的 UI 渲染等。可以理解为，每一次“大的执行单元”就是一个宏任务。
: 微任务（MicroTask）是优先级更高、粒度更小的任务，常见的包括 Promise.then / catch / finally、queueMicrotask、MutationObserver，以及 Node 中的 process.nextTick。微任务通常用于在当前宏任务结束后、尽快执行的一些回调逻辑，比如 Promise 的链式调用。
: 事件循环的核心规则是：每执行完一个宏任务，都会立即清空当前所有的微任务队列，然后浏览器进行一次渲染（如有需要），接着再进入下一轮宏任务。也就是说，微任务一定会在下一个宏任务开始前全部执行完成，这也是为什么 Promise.then 的执行顺序总是早于 setTimeout。

```javascript
// 同步 → 1
// 宏任务注册
// 微任务注册
// 同步 → 4
// 清空微任务 → 3
// 执行下一个宏任务 → 2
console.log("1");

setTimeout(() => {
  console.log("2");
}, 0);

Promise.resolve().then(() => {
  console.log("3");
});

console.log("4");
```

```javascript
// 1 2 3
// 第一个宏任务执行
// 产生微任务 → 立即清空 → 输出 2
// 再执行第二个宏任务
setTimeout(() => {
  console.log("1");
  Promise.resolve().then(() => {
    console.log("2");
  });
}, 0);

setTimeout(() => {
  console.log("3");
}, 0);
```

```javascript
// 1 3 2
// await 后面的代码进入微任务队列
// 等当前宏任务结束再执行
async function test() {
  console.log(1);
  await Promise.resolve();
  console.log(2);
}

test();
console.log(3);
```

Promise 为什么是微任务？
: Promise 被设计为微任务，是因为它需要在当前宏任务执行结束后尽快运行回调，同时又不能打断当前同步代码的执行；如果它被放入宏任务队列，就可能被后续的定时器或其他宏任务插队，从而破坏链式调用的顺序和可预测性。作为微任务，Promise.then 会在本轮宏任务结束后立即执行，并且在进入下一轮宏任务之前被全部清空，这样既保证了异步特性，又确保了链式调用的执行顺序稳定、可控。

MutationObserver
: MutationObserver 是浏览器提供的一个用于监听 DOM 结构变化的 API，可以监控节点的增删、属性变化或文本内容变动，当 DOM 发生变化时会触发回调；它的回调执行时机属于微任务队列，也就是说会在当前宏任务结束后、下一轮宏任务开始前执行，因此比 setTimeout 更早。相比早期的 DOM 事件（如 DOMSubtreeModified），MutationObserver 性能更好、可配置性更强，常用于实现数据驱动视图更新、富文本编辑器监听、或框架底层的响应式更新机制。

Promise有哪些状态？
: Promise 有三种状态：pending（初始）、fulfilled（成功）和 rejected（失败），其状态只能从 pending 变为 fulfilled 或 rejected，且一旦改变就不可逆，无法再次切换；当状态发生改变时，会触发对应的回调函数——成功时执行 then 中的成功回调，失败时执行 catch（本质也是 then 的失败回调），这也是 Promise 能够实现可预测异步流程控制的基础。

Promise的链式调用
: Promise 的链式调用本质在于 then 每次都会返回一个新的 Promise，因此可以连续调用形成链式结构；前一个 then 的返回值会作为下一个 then 的参数传入，如果返回的是普通值，会被自动包装成已成功的 Promise 继续向下传递，如果返回的是一个 Promise，则会“等待”它完成后再继续执行后续逻辑；而如果在 then 中抛出错误或返回一个 rejected 的 Promise，错误会沿着链向下传播，直到被 catch 捕获。

Value Passing
: 值穿透指的是当 then 没有传入回调函数或传入的不是函数时，Promise 会自动把上一个成功或失败的结果原样传递给下一个 then 或 catch，就像默认执行了一个“返回原值”的函数一样。

```javascript
Promise.resolve(1)
  .then()
  .then()
  .then((res) => console.log(res)); // 1
```

Error Bubbling
: Promise 中一旦在某个 then 里抛出错误或返回一个 rejected 的 Promise，这个错误就会沿着后续的链式调用向下传播，直到遇到 catch 才会被捕获；因此一个 catch 可以统一处理之前所有未被处理的错误，而从本质上说，catch 只是 then(null, onRejected) 的语法糖，用来专门处理失败回调。

```javascript
Promise.resolve()
  .then(() => {
    throw new Error("出错");
  })
  .then(() => {})
  .catch((err) => {
    console.log("捕获到错误");
  });
```

Promise 并发方法
: 常用的方法如下表

| 方法       | 成功条件 | 失败条件     | 使用场景             | 代码示例                                             |
| ---------- | -------- | ------------ | -------------------- | ---------------------------------------------------- |
| all        | 全部成功 | 任意失败     | 多请求并发，缺一不可 | `Promise.all([p1,p2]).then(res=>{}).catch(err=>{})`  |
| allSettled | 全部完成 | 不会整体失败 | 统计所有结果         | `Promise.allSettled([p1,p2]).then(res=>{})`          |
| race       | 谁先完成 | 谁先失败     | 超时控制             | `Promise.race([p1,p2]).then(res=>{}).catch(err=>{})` |
| any        | 任一成功 | 全部失败     | 只要有一个成功       | `Promise.any([p1,p2]).then(res=>{}).catch(err=>{})`  |

async/await
: async 函数的本质是始终返回一个 Promise：如果函数中 return 普通值，会被自动包装成 Promise.resolve；如果抛出错误，则会被自动转换成 Promise.reject，因此 async 本质上只是对 Promise 的一层语法封装，让异步流程以更接近同步代码的方式书写。
: await 的执行逻辑是先计算表达式，如果是 Promise 就等待其完成，如果不是就自动包装成已成功的 Promise，然后暂停当前 async 函数的后续执行，等 Promise resolve 后再继续往下执行；需要注意的是，await 只会暂停当前函数，不会阻塞整个 JavaScript 线程，其后的代码会以微任务的形式在当前宏任务结束后执行。

async/await与Generator是什么关系？
: await 能暂停函数，是因为底层借鉴了 Generator 的“yield 暂停”思想。普通函数是做不到“暂停再恢复”的，只有 Generator 可以暂停执行。yield 会暂停函数，next() 会恢复函数：

```javascript
function* gen() {
  console.log(1);
  yield;
  console.log(2);
}

const g = gen();
g.next(); // 打印 1
g.next(); // 打印 2
```

如果用 Generator 来写异步流程，比如：

```javascript
function* gen() {
  const a = yield fetchA();
  const b = yield fetchB(a);
  return b;
}
```

每次执行到 yield 都会暂停函数，并返回一个 Promise，但 Generator 本身并不会自动等待 Promise 完成，也不会在 resolve 后自动继续执行，因此必须有一个“自动执行器”不断调用 next()：先执行一次 next() 拿到 Promise，等待它 resolve，再把结果传回 next(结果) 继续执行，如此循环直到结束；这个负责“等待 + 继续”的调度器在社区中常见实现叫 co。

async/await 本质上就是把“Generator + 自动执行器 + Promise”这套机制封装成语法糖：当你写：

```javascript
async function fn() {
  const a = await fetchA();
  const b = await fetchB(a);
  return b;
}
```

底层相当于把函数转成类似 Generator 的可暂停结构，并自动创建执行器，遇到 await 就暂停，等 Promise resolve 后自动恢复执行，只是这些 next() 和 then() 的控制流程都由引擎替你完成了。

Minimum Delay
: setTimeout(fn, 0) 并不意味着“立刻执行”，它只是把回调放入宏任务队列，等当前宏任务执行结束后尽快调度；而且浏览器对定时器存在最小延时限制，根据规范，当连续嵌套超过 5 层时，最小延时会被强制设为 4ms，因此即使传入 0，也可能实际延迟 4ms 才执行。
: 此外，在浏览器后台标签页中，定时器通常会被降频处理，延时可能被限制到 1000ms 甚至更高，这是浏览器出于性能和节能考虑的策略；因此需要理解的是，setTimeout 指定的时间并不是“精确执行时间”，而只是“最早可执行时间”，真正执行还必须等调用栈清空、前序任务完成并满足最小延时条件后才会发生。

递归定时
: 所谓“递归定时”是指在一次 setTimeout 回调执行结束时再开启下一次定时，这种方式相比 setInterval 更安全，因为 setInterval 是按固定时间间隔不断把任务加入队列，如果回调执行时间超过设定间隔，就会出现任务堆积、连续触发的情况；而递归 setTimeout 是“执行完一次再调度下一次”，保证同一时间只存在一个待执行任务，从而避免堆积问题。

```js
setTimeout(function fn() {
  console.log("tick");
  setTimeout(fn, 0);
}, 0);
```

如何实现精确计时？
: 实现“精确计时”的核心原理是：不要依赖定时器本身的固定间隔，因为定时器并不能保证严格按设定时间执行，它只能保证“最早在这个时间之后执行”，如果某一次执行被阻塞或延迟，误差就会不断累积，最终导致整体节奏越来越慢。
: 更准确的做法是以“真实时间”为基准进行校准：每次执行时都对比当前时间和理论上应该到达的时间之间的差值，如果发现晚了，就在下一次调度时适当缩短等待时间进行补偿。这样做的本质是通过持续校正时间漂移，避免误差累加，使整体节奏始终围绕目标时间点波动，而不是越跑越偏。

```js
let start = Date.now();

function loop() {
  const drift = Date.now() - start - 1000;
  start = Date.now();
  setTimeout(loop, 1000 - drift);
}
```

手写Promise基础实现
: 待实现

手写Promise.all并发控制limit
: 待实现

## 数据结构

数组初始化
: 二维写法为 `Array.from({ length: m }, () => new Array(n).fill(0))` ，一维见表

| 需求              | 写法                                | 说明                  | 是否推荐       |
| ----------------- | ----------------------------------- | --------------------- | -------------- |
| 长度为 n 的空数组 | `new Array(n)`                      | 只有 length，没有元素 | ⚠ 不推荐直接用 |
| 长度为 n 全 0     | `new Array(n).fill(0)`              | 所有元素为 0          | ✅ 常用        |
| 长度为 n 全 1     | `new Array(n).fill(1)`              | 所有元素为 1          | ✅ 常用        |
| 生成 0 ~ n-1      | `Array.from({length:n},(_,i)=>i)`   | 最常用生成序列        | ⭐⭐⭐ 推荐    |
| 生成 1 ~ n        | `Array.from({length:n},(_,i)=>i+1)` | 序列变形              | ⭐⭐           |
| 拷贝数组          | `[...arr]`                          | 浅拷贝                | ⭐⭐           |

数组常用方法
: 见表

| 分类      | 方法      | 是否改变原数组 | 核心作用                 | 典型场景    |
| --------- | --------- | -------------- | ------------------------ | ----------- |
| 栈操作    | push      | ✅             | 末尾添加                 | 模拟栈      |
| 栈操作    | pop       | ✅             | 末尾删除                 | 单调栈      |
| 队列操作  | shift     | ✅             | 头部删除                 | 队列        |
| 队列操作  | unshift   | ✅             | 头部添加                 | 双端队列    |
| 截取      | slice     | ❌             | 返回子数组               | 拷贝 / 分割 |
| 删除/插入 | splice    | ✅             | 删除/替换/插入           | 修改数组    |
| 合并      | concat    | ❌             | 合并数组                 | 合并结果    |
| 查找      | includes  | ❌             | 是否存在                 | 存在判断    |
| 查找      | indexOf   | ❌             | 查找下标                 | 找位置      |
| 查找      | find      | ❌             | 找到第一个满足条件元素   | 条件查找    |
| 查找      | findIndex | ❌             | 找到第一个满足条件下标   | 条件查找    |
| 判断      | some      | ❌             | 是否存在满足条件         | 至少一个    |
| 判断      | every     | ❌             | 是否全部满足             | 全部判断    |
| 映射      | map       | ❌             | 映射新数组               | 转换数据    |
| 过滤      | filter    | ❌             | 过滤元素                 | 条件筛选    |
| 聚合      | reduce    | ❌             | 累加/统计                | 计数/求和   |
| 排序      | sort      | ✅             | 排序                     | 排序题      |
| 反转      | reverse   | ✅             | 反转数组                 | 双指针      |
| 转字符串  | join      | ❌             | 拼接字符串               | 输出格式    |
| 访问      | at        | ❌             | 访问指定索引（支持负数） | 取尾元素    |

字符串常用方法
: 见表

| 方法        | 核心作用                   | 示例                                  |
| ----------- | -------------------------- | ------------------------------------- |
| split       | 按分隔符拆分为数组         | `"a,b,c".split(",") // ["a","b","c"]` |
| substring   | 截取子字符串（不支持负数） | `"hello".substring(1,4) // "ell"`     |
| slice       | 截取子字符串（支持负数）   | `"hello".slice(-2) // "lo"`           |
| toLowerCase | 转小写                     | `"ABC".toLowerCase() // "abc"`        |
| trim        | 去除两端空格               | `"  hi  ".trim() // "hi"`             |
| indexOf     | 查找子串位置               | `"hello".indexOf("l") // 2`           |
| replace     | 替换子串                   | `"aabb".replace("a","x") // "xabb"`   |
| charCodeAt  | 获取字符 Unicode 编码      | `"a".charCodeAt(0) // 97`             |
| includes    | 是否包含子串               | `"hello".includes("he") // true`      |
| startsWith  | 是否以某字符串开头         | `"hello".startsWith("he") // true`    |
| endsWith    | 是否以某字符串结尾         | `"hello".endsWith("lo") // true`      |
| repeat      | 重复字符串                 | `"ha".repeat(3) // "hahaha"`          |
| padStart    | 头部补齐                   | `"5".padStart(3,"0") // "005"`        |
| padEnd      | 尾部补齐                   | `"5".padEnd(3,"0") // "500"`          |
| str[idx]    | 访问指定位置字符           | `"hello"[1] // "e"`                   |

对象操作
: 见表

| 方法           | 核心作用                       | 示例                                  |
| -------------- | ------------------------------ | ------------------------------------- |
| Object.keys    | 获取对象自身可枚举属性名数组   | `Object.keys({a:1,b:2}) // ["a","b"]` |
| Object.values  | 获取对象自身可枚举属性值数组   | `Object.values({a:1,b:2}) // [1,2]`   |
| Object.entries | 获取对象自身可枚举键值对数组   | `Object.entries({a:1}) // [["a",1]]`  |
| hasOwnProperty | 判断是否为对象自身属性         | `obj.hasOwnProperty("a")`             |
| for...in       | 遍历对象可枚举属性（含原型链） | `for (let key in obj) {}`             |
| in             | 判断属性是否存在（含原型链）   | `"a" in obj`                          |

Map 操作
: 见表

| 方法 / 属性     | 作用           | 示例                        |
| --------------- | -------------- | --------------------------- |
| new Map()       | 创建 Map       | `const map = new Map()`     |
| set(key, value) | 设置键值对     | `map.set("a", 1)`           |
| get(key)        | 获取值         | `map.get("a") // 1`         |
| has(key)        | 判断是否存在   | `map.has("a") // true`      |
| delete(key)     | 删除键         | `map.delete("a")`           |
| clear()         | 清空所有       | `map.clear()`               |
| size            | 获取大小       | `map.size`                  |
| keys()          | 获取所有 key   | `map.keys()`                |
| values()        | 获取所有 value | `map.values()`              |
| entries()       | 获取键值对     | `map.entries()`             |
| forEach()       | 遍历           | `map.forEach((v,k)=>{})`    |
| for...of        | 遍历键值对     | `for (let [k,v] of map) {}` |

Set 操作
: 见表

| 方法 / 属性   | 作用               | 示例                    |
| ------------- | ------------------ | ----------------------- |
| new Set()     | 创建 Set           | `const set = new Set()` |
| add(value)    | 添加元素           | `set.add(1)`            |
| has(value)    | 是否存在           | `set.has(1)`            |
| delete(value) | 删除元素           | `set.delete(1)`         |
| clear()       | 清空               | `set.clear()`           |
| size          | 获取大小           | `set.size`              |
| values()      | 获取所有值         | `set.values()`          |
| keys()        | 与 values 相同     | `set.keys()`            |
| entries()     | 获取 [value,value] | `set.entries()`         |
| forEach()     | 遍历               | `set.forEach(v=>{})`    |
| for...of      | 遍历值             | `for (let v of set) {}` |

WeakMap/WeakSet的作用是什么？
: WeakMap 和 WeakSet 用于存储对象的弱引用数据，当对象没有其他引用时会自动被垃圾回收，因此常用于缓存或私有数据存储以避免内存泄漏。

```javascript
// 当实例被销毁：WeakMap 里的数据也会自动释放
const privateData = new WeakMap();

class User {
  constructor(name) {
    privateData.set(this, { name });
  }

  getName() {
    return privateData.get(this).name;
  }
}
```
