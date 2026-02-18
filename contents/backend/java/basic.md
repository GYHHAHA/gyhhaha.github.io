---
short_title: Java Basic
---

# Java 基础

## 基本概念

### var关键词

var 是局部变量类型推断（local variable type inference），它的意思不是“动态类型”，而是让编译器根据右边的表达式自动推断变量类型，只允许用于“方法内部 + 有明确初始化表达式”的局部变量，且方法签名必须显式声明，不能隐式推断。

| 场景                    | 能否使用 `var` | 示例                                             |
| ----------------------- | -------------- | ------------------------------------------------ |
| 方法内部局部变量        | ✅             | `var x = 10;`                                    |
| 增强 for 循环变量       | ✅             | `for (var item : list) {}`                       |
| 普通 for 循环变量       | ✅             | `for (var i = 0; i < 10; i++) {}`                |
| try-with-resources      | ✅             | `try (var in = new FileInputStream("a.txt")) {}` |
| Lambda 参数（Java 11+） | ✅             | `(var x, var y) -> x + y`                        |
| 成员变量（字段）        | ❌             | `class A { var x = 10; }`                        |
| 方法参数                | ❌             | `void test(var x) {}`                            |
| 构造器参数              | ❌             | `A(var x) {}`                                    |
| 方法返回类型            | ❌             | `public var test() {}`                           |
| 未初始化变量            | ❌             | `var x;`                                         |
| 初始化为 null           | ❌             | `var x = null;`                                  |
| 数组字面量（无 new）    | ❌             | `var arr = {1,2,3};`                             |
| 数组（带 new）          | ✅             | `var arr = new int[]{1,2,3};`                    |

### 基本类型

| 类型      | 大小                 | 默认值（成员变量） | 取值范围             |
| --------- | -------------------- | ------------------ | -------------------- |
| `byte`    | 1 字节               | 0                  | -128 ~ 127           |
| `short`   | 2 字节               | 0                  | -32,768 ~ 32,767     |
| `int`     | 4 字节               | 0                  | -2³¹ ~ 2³¹-1         |
| `long`    | 8 字节               | 0L                 | -2⁶³ ~ 2⁶³-1         |
| `float`   | 4 字节               | 0.0f               | ±1.4E-45 ~ ±3.4E38   |
| `double`  | 8 字节               | 0.0d               | ±4.9E-324 ~ ±1.7E308 |
| `char`    | 2 字节               | '\u0000'           | 0 ~ 65535            |
| `boolean` | JVM相关（通常1字节） | false              | true / false         |

### 包装类

自动装箱（Autoboxing）是指 Java 编译器在基本数据类型和其对应的包装类之间进行的自动转换机制，例如将 int 自动转换为 Integer。当把基本类型赋值给包装类，或在集合中使用基本类型时，编译器会自动调用对应的 valueOf() 方法完成转换；反过来，把包装类赋值给基本类型时，会自动调用如 intValue() 这样的方法完成拆箱。

| 对比项             | 基本类型 (primitive) | 包装类 (wrapper) |
| ------------------ | -------------------- | ---------------- |
| 是否是对象         | ❌ 不是              | ✅ 是对象        |
| 是否有方法         | ❌ 没有              | ✅ 有方法        |
| 是否可以为 null    | ❌ 不可以            | ✅ 可以          |
| 存储位置           | 栈内存（局部变量）   | 堆内存           |
| 默认值（成员变量） | 0 / false            | null             |
| 是否支持泛型       | ❌ 不支持            | ✅ 支持          |
| 性能               | 更高                 | 略低（对象开销） |
| 可用于集合         | ❌ 不行              | ✅ 可以          |

### 字符串

String 的不可变性指的是：一旦 String 对象被创建，它的内容就不能被修改。任何看似“修改”字符串的操作（例如拼接、replace 等）都会创建一个新的 String 对象，而不是改变原有对象的内容。正因为这种不可变特性，String 才能够实现线程安全、支持字符串常量池复用、并且可以安全地作为 HashMap 的 key 使用。

String 的常见操作：

| 分类   | 方法                    | 作用               | 常见用途         |
| ------ | ----------------------- | ------------------ | ---------------- |
| 长度   | `length()`              | 获取长度           | 双指针、边界判断 |
| 取字符 | `charAt(i)`             | 获取指定位置字符   | 遍历字符串       |
| 转数组 | `toCharArray()`         | 转成 char[]        | 原地修改、排序   |
| 子串   | `substring(begin, end)` | 截取子串           | 滑动窗口         |
| 比较   | `equals()`              | 比较内容           | 判等             |
|        | `equalsIgnoreCase()`    | 忽略大小写         | 字符串匹配       |
| 查找   | `indexOf()`             | 查找字符/子串位置  | 子串问题         |
|        | `lastIndexOf()`         | 从后查找           | 反向匹配         |
| 判断   | `contains()`            | 是否包含子串       | 子串存在判断     |
|        | `startsWith()`          | 是否以某字符串开头 | 前缀判断         |
|        | `endsWith()`            | 是否以某字符串结尾 | 后缀判断         |
| 替换   | `replace()`             | 替换字符/字符串    | 字符替换         |
|        | `replaceAll()`          | 正则替换           | 模式替换         |
| 分割   | `split()`               | 按分隔符拆分       | 字符串解析       |
| 去空格 | `trim()`                | 去首尾空格         | 输入清洗         |
| 拼接   | `concat()`              | 拼接字符串         | 少用，常用 +     |
| 大小写 | `toLowerCase()`         | 转小写             | 忽略大小写题     |
|        | `toUpperCase()`         | 转大写             | 规范化处理       |
| 转数字 | `valueOf()`             | 基本类型转字符串   | 输出             |
|        | `Integer.parseInt()`    | 字符串转 int       | 数字题           |

String 类上的静态方法：

| 方法                   | 作用                       | 常见用途         |
| ---------------------- | -------------------------- | ---------------- |
| `String.valueOf()`     | 将基本类型或对象转为字符串 | 输出转换         |
| `String.format()`      | 格式化字符串               | 模板拼接         |
| `String.join()`        | 用分隔符拼接多个字符串     | 拼接列表         |
| `String.copyValueOf()` | 将 char[] 转为 String      | 字符数组转字符串 |

````{tip} 格式化输出
`String.format()` 方法用于格式化字符串，可以将多个参数插入到字符串中，以便生成更复杂的输出。

```java
String name = "Tom";
int age = 18;

System.out.println(
    String.format("My name is %s, age is %d", name, age)
);
```

常用的占位符有：

| 占位符    | 含义     |
| ------ | ------ |
| `%s`   | 字符串    |
| `%d`   | 整数     |
| `%f`   | 浮点     |
| `%.2f` | 保留两位小数 |
| `%n`   | 换行     |
````

StringBuilder 是一个可变字符串类，String 不可变，StringBuilder 可修改，不会每次创建新对象，它主要用于频繁拼接字符串和构造字符串。

| 方法                       | 作用         | 常见用途   |
| -------------------------- | ------------ | ---------- |
| `append()`                 | 追加内容     | 拼接字符串 |
| `insert(index, str)`       | 指定位置插入 | 构造字符串 |
| `delete(start, end)`       | 删除区间     | 删除字符   |
| `deleteCharAt(index)`      | 删除单个字符 | 回溯       |
| `replace(start, end, str)` | 替换区间     | 字符串修改 |
| `reverse()`                | 反转字符串   | 回文题     |
| `charAt(index)`            | 获取字符     | 读取       |
| `setCharAt(index, ch)`     | 修改字符     | 原地修改   |
| `length()`                 | 当前长度     | 边界控制   |
| `capacity()`               | 当前容量     | 性能优化   |
| `toString()`               | 转为 String  | 返回结果   |

````{tip} 多行字符串
Text Block 是 Java 15 引入的多行字符串语法，会自动去掉最小公共缩进。

```java
String str = """
    line 1
    line 2
    line 3
    """;
```
````

### 数组

Java 数组在创建时必须指定长度，且一旦创建，长度不可变。

| 方法             | 作用             | 常见用途               |
| ---------------- | ---------------- | ---------------------- |
| `sort()`         | 排序             | 排序数组（算法题高频） |
| `binarySearch()` | 二分查找         | 已排序数组查找         |
| `toString()`     | 打印一维数组     | 调试输出               |
| `deepToString()` | 打印多维数组     | 二维数组调试           |
| `equals()`       | 比较一维数组内容 | 判断数组相等           |
| `deepEquals()`   | 比较多维数组     | 二维数组比较           |
| `fill()`         | 填充数组         | 初始化                 |
| `copyOf()`       | 复制数组         | 扩容/截断              |
| `copyOfRange()`  | 复制指定区间     | 子数组                 |
| `asList()`       | 数组转 List      | 集合转换               |
| `stream()`       | 数组转 Stream    | 函数式处理             |
| `parallelSort()` | 并行排序         | 大数组优化             |

### 内存模型

- 栈内存（Stack）：存放方法调用时的栈帧、局部变量、基本类型的值以及对象引用（不存对象本身）；线程私有，方法执行结束自动释放，分配和回收速度快，不由 GC 管理。
- 堆内存（Heap）：存放通过 new 创建的对象实例和数组；线程共享，生命周期由 GC 决定，内存占用最大，访问相对栈慢，但灵活性高。
- 常量池（Constant Pool，常见指字符串常量池）：存放编译期确定的常量、字符串字面量以及被 intern() 的字符串；全局共享，用于避免重复创建相同常量，提高内存利用率。

| 对比项       | 栈内存（Stack）        | 堆内存（Heap） | 常量池（Constant Pool） |
| ------------ | ---------------------- | -------------- | ----------------------- |
| 存储内容     | 局部变量、方法调用信息 | 对象实例       | 常量、字符串常量        |
| 是否线程共享 | ❌ 不共享（线程私有）  | ✅ 共享        | ✅ 共享                 |
| 生命周期     | 随方法结束释放         | 由 GC 回收     | 随类加载存在            |
| 存储数据类型 | 基本类型值、对象引用   | 对象本身       | 字面量常量              |
| 是否有 GC    | ❌ 无                  | ✅ 有          | 部分由 GC 管理          |
| 访问速度     | 快                     | 相对慢         | 快                      |

## 流程控制

### 增强 for 循环

增强 for 循环是 Java 5 引入的语法糖，用于 简化对数组或集合的遍历。

```java
for (元素类型 变量 : 集合或数组) {
    // 使用变量
}
```

增强 for 循环本质上是：

- 对数组：基于索引循环
- 对集合：使用 Iterator

```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
}
```

### case 语句

传统 switch 必须写 break，否则会 fall-through：

```java
int day = 2;

switch (day) {
    case 1:
        System.out.println("Monday");
        break;
    case 2:
        System.out.println("Tuesday");
        break;
    default:
        System.out.println("Other");
}
```

Java 14 引入了 switch expression，不需要写 break，不会 fall-through

```java
switch (day) {
    case 1 -> System.out.println("Monday");
    case 2 -> System.out.println("Tuesday");
    default -> System.out.println("Other");
}
```

此外，可以有返回值：

```java
String result = switch (day) {
    case 1, 2 -> "Workday";
    default -> "Other";
};
```

复杂逻辑用 yield：

```java
String result = switch (day) {
    case 1 -> {
        System.out.println("Processing...");
        yield "Monday";
    }
    default -> "Other";
};
```

## 异常处理

### 异常的结构

Java 所有异常都继承自 java.lang.Throwable，结构如下

```
Throwable
├── Error // 表示 JVM 级别错误，一般程序无法恢复，不建议捕获
└── Exception
      ├── RuntimeException // unchecked exception
      └── 其他受检异常（Checked Exception） // 必须显式处理（try-catch 或 throws）
```

### 异常的处理

1. try-catch

```java
try {
    int a = 10 / 0;
} catch (ArithmeticException e) {
    System.out.println("Error");
}
```

2. throws

```java
public void readFile() throws IOException {
}
```

```{tip} throw和throws的区别
| 关键字      | 位置   | 作用              | 例子                                  |
| -------- | ---- | --------------- | ----------------------------------- |
| `throws` | 方法名后 | 声明：预告该方法可能抛出的异常 | `void test() throws Exception`      |
| `throw`  | 方法体内 | 动作：主动创建并抛出一个异常  | `throw new RuntimeException("错啦");` |

```

3. finally

```java
try {
    ...
} catch (...) {
    ...
} finally {
    // 一定执行
}
```

````{tip} try-catch with resource
在 Java 7 之前，关闭资源（如文件流、数据库连接）简直是开发者的噩梦。你必须在 finally 块里手动关闭，而且为了防止关闭本身也出异常，还得在 finally 里面再嵌套一个 try-catch，代码极其臃肿。

```java
BufferedReader br = null;
try {
    br = new BufferedReader(new FileReader("test.txt"));
    System.out.println(br.readLine());
} catch (IOException e) {
    e.printStackTrace();
} finally {
    // 这里的关闭逻辑非常恶心
    if (br != null) {
        try {
            br.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

try-with-resources 的核心原理是基于 java.lang.AutoCloseable 接口：只有实现了该接口（或其子接口 java.io.Closeable）的类，才能写在 try(...) 括号中，并在代码执行结束后被自动调用 close() 方法进行资源释放。因此，Java 标准库中的 IO 流、数据库连接、网络 Socket 等资源类都实现了这个接口，从而支持自动关闭，避免资源泄漏。

```java
try (BufferedReader br = new BufferedReader(new FileReader("test.txt"))) {
    System.out.println(br.readLine());
} catch (IOException e) {
    e.printStackTrace();
}
// 不需要写 finally，br 会被自动关闭
```
````
