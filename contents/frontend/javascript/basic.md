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

var 变量提升
: var 声明的变量会将声明部分提升到作用域顶部，但赋值留在原地，因此在赋值前访问会得到 undefined。

函数提升
: 使用 function 关键字声明的函数会将整个函数体提升到作用域顶部，这使得函数可以在声明语句之前被安全调用。

箭头函数（Arrow Function）与普通函数（Regular Function）的区别？
: 见下表

| 特性         | 普通函数（function）               | 箭头函数（=>）                                  |
| ------------ | ---------------------------------- | ----------------------------------------------- |
| this 指向    | 动态绑定：指向调用它的那个对象。   | 静态绑定：继承自定义时外层作用域的 this。       |
| 构造函数     | 可以作为构造函数（有 prototype）。 | 不能作为构造函数（无 prototype），会报错。      |
| arguments    | 可直接访问实参列表对象。           | 无自己的 arguments，需用 ...args 替代。         |
| 变量提升     | 会整体提升。                       | 作为变量赋值，遵循 let/const 的规则（有死区）。 |
| yield 关键字 | 可以作为 Generator 函数。          | 不能作为 Generator 函数。                       |

闭包
: 闭包是一个使函数能够记住并访问其词法作用域的特性，即使该函数在当前词法作用域之外执行。

1. 循环 + 定时器

```javascript
/***********************
 * ① 原始问题代码
 * 结果：1秒后连续输出 5 次 5
 ***********************/
for (var i = 0; i < 5; i++) {
  // setTimeout 是异步任务，会在当前同步代码执行完后再执行
  setTimeout(() => {
    console.log(i); // 此时循环早已结束，i 已经变成 5
  }, 1000);
}
// 原因：var 没有块级作用域，i 是全局共享变量

/***********************
 * ② ES5 闭包解法（IIFE）
 * 结果：1秒后输出 0 1 2 3 4
 ***********************/
for (var i = 0; i < 5; i++) {
  (function (j) {
    // 这里的 j 是函数的私有变量（形成闭包）
    // 每次循环都会把当前 i 传进来并“锁住”
    setTimeout(() => {
      console.log(j); // 输出的是各自独立保存的 j
    }, 1000);
  })(i); // 立即执行，并把当前 i 作为参数传入
}

/***********************
 * ③ 现代解法（ES6 let）
 * 结果：1秒后输出 0 1 2 3 4
 ***********************/
for (let i = 0; i < 5; i++) {
  // let 具有块级作用域
  // 每次循环都会创建一个新的 i（独立的词法环境）
  setTimeout(() => {
    console.log(i); // 访问的是当前那一轮循环的 i
  }, 1000);
}
```

2. 私有变量

```javascript
function createCounter() {
  let count = 0; // 私有变量，外部无法直接访问
  return {
    add: () => ++count,
    get: () => count,
  };
}
const counter = createCounter();
console.log(counter.add()); // 1
```

防抖 (Debounce) 和 节流 (Throttle)
: 区别见下

| 技术 | 核心逻辑                   | 常见应用场景                                                                                  |
| ---- | -------------------------- | --------------------------------------------------------------------------------------------- |
| 防抖 | 只要你一直在动，我就不动。 | 1. 搜索框输入查询（用户打完字再请求）。2. 窗口大小调整（resize）（调整停止后再计算尺寸）。    |
| 节流 | 不管你动多快，我按节奏动。 | 1. 滚动加载（scroll）（滑到底部触发加载）。2. 高频点击提交（防止重复下单）。3. 抢购按钮点击。 |

深拷贝与浅拷贝
: 见下

| 方法            | 拷贝深度 | 循环引用支持 | 特殊对象 (Date/Reg) | 函数支持 |
| --------------- | -------- | ------------ | ------------------- | -------- |
| 展开运算符      | 浅       | N/A          | 原样引用            | 支持     |
| JSON 方法       | 深       | 报错         | 失效                | 丢失     |
| structuredClone | 深       | 支持         | 支持                | 不支持   |
| 手写 + WeakMap  | 深       | 支持         | 可定制              | 支持     |

默认参数
: ES6 引入，解决了以往通过 a = a || 10 这种写法可能遇到的坑（比如传 0 或 false 时会被误判）。默认值只有在参数缺失或为 undefined 时才执行。参数名存在于自己的作用域中，后面的参数默认值可以引用前面的参数，反之则报错。

```javascript
function multiply(a, b = 2) { return a * b; }
function foo(a = b, b = 1) { ... } // 报错！b 在使用时还未定义
```

剩余参数 (Rest Parameters) vs arguments
: 见下表

| 特性     | 剩余参数（...args）          | arguments 对象                     |
| -------- | ---------------------------- | ---------------------------------- |
| 类型     | 真正的数组。                 | 类数组对象（只有 length 和索引）。 |
| 包含内容 | 仅包含未命名的剩余参数。     | 包含函数接收到的所有参数。         |
| 箭头函数 | 支持。                       | 不支持。                           |
| 解构支持 | 支持，必须放在参数列表最后。 | 不支持。                           |

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

## 原型链

### 核心概念

三个角色：

1. `prototype` (显式原型)：每一个函数（严格来说是构造函数）都有一个 `prototype` 属性。它指向一个对象，这个对象包含了所有实例共享的方法和属性。
2. `__proto__` (隐式原型)：每一个对象（包括函数）都有一个 `__proto__` 属性。它指向创建该对象的构造函数的 `prototype`。
3. `constructor` (构造函数)：每个原型对象都有一个 `constructor` 属性，指向它关联的那个构造函数。

当你访问一个对象的属性时（比如 `obj.a`）：

1. 先在 `obj` 自身寻找。
2. 找不到，就顺着 `obj.__proto__`（即它的构造函数的 `prototype`）去找。
3. 如果还找不到，就继续往 `obj.__proto__.__proto__` 找。
4. 一直找到 `Object.prototype.__proto__`，也就是 `null` 为止。

这就是原型链。如果整条链都找不着，返回 `undefined`。

`__proto__` 指向问题：

1. `对象的.__proto__ === 其构造函数.prototype`
2. `函数的.__proto__ === Function.prototype` (因为函数也是由 `Function` 构造的)

特殊情况：

1. `Object.prototype.__proto__ === null` (原型链的尽头)
2. `Function.prototype.__proto__ === Object.prototype` (函数原型本质也是对象)

手写 `instanceof`

```javascript
function myInstanceof(left, right) {
  let proto = Object.getPrototypeOf(left); // 拿到左边的隐式原型
  let prototype = right.prototype; // 拿到右边的显式原型

  while (true) {
    if (!proto) return false; // 找到尽头 null 了还没找到
    if (proto === prototype) return true; // 找到了！
    proto = Object.getPrototypeOf(proto); // 继续向上爬链
  }
}
```

### 继承方式

1. 原型链继承

核心：`Child.prototype = new Parent()`

- 原理：直接将子类的原型指向父类的实例。
- 缺点：
  - 引用属性共享：如果父类有一个数组属性，一个子类实例修改了它，所有实例都会跟着变。
  - 无法传参：创建子类实例时，无法向父类构造函数传参。

2. 构造函数继承 (借助 `call`)

核心：在子类构造函数中执行 `Parent.call(this, ...args)`

- 原理：将父类的属性“复制”到子类实例上。
- 优点：解决了引用共享和传参问题。
- 缺点：函数无法复用。每个实例都会创建一份父类方法的副本，没用到原型链，极其浪费内存。

3. 组合继承 (最常用/最稳妥)

核心：原型链继承（继承方法）+ 构造函数继承（继承属性）。

```javascript
function Child(name) {
  Parent.call(this, name); // 第二次调用 Parent
}
Child.prototype = new Parent(); // 第一次调用 Parent
```

- 缺点：父类构造函数会被调用两次，子类原型上会存在一份多余的、无用的父类属性。

4. 寄生组合继承 (面试“满分”答案) ⭐

核心：不直接 `new Parent()`，而是通过 `Object.create(Parent.prototype)` 创建一个中间空对象。

```javascript
function inherit(Child, Parent) {
  // 创建一个以 Parent.prototype 为原型的空对象，赋给子类原型
  Child.prototype = Object.create(Parent.prototype);
  // 修正 constructor 指向
  Child.prototype.constructor = Child;
}
```

- 优点：只调用一次父类构造函数，原型链保持干净，是 ES6 之前最完美的继承方案。

5. ES6 `class` 继承 (`extends` & `super`)

核心：语法糖，底层依然是基于原型链。

```javascript
class Child extends Parent {
  constructor(name, age) {
    super(name); // 必须在使用 this 之前调用 super，相当于 Parent.call(this)
    this.age = age;
  }
}
```

ES5 和 ES6 继承的区别？

- `this` 的生成顺序不同：
  - ES5：先创建子类的 `this`，再通过 `Parent.call(this)` 把父类的属性加到子类 `this` 上。
  - ES6：先由 `super()` 创建父类的实例 `this`，然后再用子类的构造函数去修改这个 `this`。这也是为什么 `class` 继承必须先写 `super()`。
- 静态属性继承：ES6 的 `extends` 不仅继承原型，还会继承父类的静态方法（即 `Child.__proto__ === Parent`）。

| 方式         | 核心机制                         | 评价                     |
| ------------ | -------------------------------- | ------------------------ |
| 原型链       | `Child.prototype = new Parent()` | 会篡改引用属性，不推荐。 |
| 借用构造函数 | `Parent.call(this)`              | 方法无法复用，性能差。   |
| 组合继承     | 逻辑组合                         | 功能全，但有冗余开销。   |
| 寄生组合     | `Object.create`                  | ES5 环境下的最优解。     |
| ES6 class    | `extends`                        | 现代开发标准。           |

### new 操作符

当我们在执行 `const p = new Person('Tom')` 时，JS 引擎在后台悄悄做了这四件事：

1. 创建一个新对象：在内存中开辟一块空间。
2. 关联原型：将新对象的 `__proto__` 指向构造函数的 `prototype` 属性。
3. 绑定 this 并执行：执行构造函数，并将 `this` 绑定到这个新对象上（这样构造函数里的 `this.name = name` 才能生效）。
4. 返回新对象：如果构造函数没有显式返回对象，则默认返回这个新对象。

```javascript
function myNew(constructor, ...args) {
  // 1. 创建一个空的简单 JavaScript 对象
  const obj = {};

  // 2. 将该对象的原型原型链接到构造函数的 prototype 属性
  // 等价于 obj.__proto__ = constructor.prototype
  Object.setPrototypeOf(obj, constructor.prototype);

  // 3. 将步骤 1 新创建的对象作为 this 的上下文，执行构造函数
  const result = constructor.apply(obj, args);

  // 4. 如果构造函数返回的是对象，则返回该对象；否则返回新创建的对象
  return typeof result === "object" && result !== null ? result : obj;
}
```

### this 绑定

按优先级从低到高排列：

1. 默认绑定 (Default Binding)

场景：函数独立调用，不带任何修饰。

- 非严格模式：指向 `window` (浏览器) 或 `global` (Node)。
- 严格模式 (`'use strict'`)：指向 `undefined`。

```javascript
function foo() {
  console.log(this);
}
foo(); // window
```

2. 隐式绑定 (Implicit Binding)

场景：函数作为对象的方法被调用。

- 规则：`this` 指向调用它的那个对象（最近的一层）。
- 隐式丢失：如果把对象的方法赋值给一个变量再调用，`this` 会丢失，退化为默认绑定。

```javascript
const obj = {
  name: "Gemini",
  foo() {
    console.log(this.name);
  },
};
obj.foo(); // 'Gemini'

const bar = obj.foo;
bar(); // undefined (隐式丢失)
```

3. 显式绑定 (Explicit Binding)

场景：通过 `call`、`apply`、`bind` 硬性指定 `this`。

- `call(obj, p1, p2)`：立即执行，参数平铺。
- `apply(obj, [p1, p2])`：立即执行，参数打包成数组。
- `bind(obj)`：不立即执行，返回一个永久绑定了 `this` 的新函数。

4. `new` 绑定

场景：通过 `new` 关键字调用构造函数。

- 规则：`this` 指向 `new` 出来的那个新对象。
- 优先级高于显式绑定（你可以 `bind` 一个函数后再 `new` 它，`this` 会指向 `new` 出来的实例）。

5. 箭头函数 (ES6)

它是 `this` 规则里的“法外之徒”：

- 没有自己的 `this`：它会捕获定义时所处的外层词法环境的 `this`。
- 无法被改变：`call`、`apply`、`bind` 对箭头函数无效。
- 不能用作构造函数：因为没有 `this` 绑定机制。

6. 优先级总结与判定逻辑

- 是箭头函数吗？ -> 找它外层的普通函数 `this`。
- 是 `new` 调用吗？ -> 指向新实例对象。
- 用了 `call/apply/bind` 吗？ -> 指向指定的第一个参数。
- 是对象调用吗（如 `obj.foo()`）？ -> 指向 `obj`。
- 以上都不是？ -> 严格模式下 `undefined`，非严格模式 `window`。

### call/apply/bind

| 方法  | 传递参数的方式                       | 执行时机   | 返回值         |
| ----- | ------------------------------------ | ---------- | -------------- |
| call  | 参数列表：`fn.call(obj, 1, 2)`       | 立即执行   | 函数执行的结果 |
| apply | 数组/类数组：`fn.apply(obj, [1, 2])` | 立即执行   | 函数执行的结果 |
| bind  | 参数列表：`fn.bind(obj, 1, 2)`       | 不立即执行 | 返回一个新函数 |

手写 `call`：利用“隐式绑定”。将函数设为对象的一个属性，执行后再删除该属性。

```javascript
Function.prototype.myCall = function (context, ...args) {
  // 1. 处理 context。如果不传或传 null，指向 window
  context = context || window;

  // 2. 将当前函数（this）作为 context 的一个属性
  // 使用 Symbol 防止属性重名覆盖
  const fnKey = Symbol();
  context[fnKey] = this;

  // 3. 执行函数并获取结果
  const result = context[fnKey](...args);

  // 4. 删除临时属性并返回结果
  delete context[fnKey];
  return result;
};
```

手写 `apply`：与 `call` 基本一致，区别在于处理参数的方式。

```javascript
Function.prototype.myApply = function (context, argsArray) {
  context = context || window;
  const fnKey = Symbol();
  context[fnKey] = this;

  // 判断是否有参数数组传入
  const result = Array.isArray(argsArray)
    ? context[fnKey](...argsArray)
    : context[fnKey]();

  delete context[fnKey];
  return result;
};
```

手写 `bind` ：返回一个闭包函数。注意要处理两个地方的参数：`bind` 时的参数和执行新函数时的参数。

```javascript
Function.prototype.myBind = function (context, ...args) {
  const self = this; // 保存原函数

  return function (...newArgs) {
    // 拼接 bind 时的参数和调用时的参数
    return self.apply(context, args.concat(newArgs));
  };
};
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
