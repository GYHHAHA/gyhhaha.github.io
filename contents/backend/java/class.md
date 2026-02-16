---
short_title: Class & Object
---

# 面向对象

## 基本概念

```{tip} Java 中的类
一个 `.java` 文件中，最多只能有一个 public 顶级类，文件名必须和这个 public 类名一致。Java 中不能有独立函数，函数必须属于类。
```

### 三大特性

- 封装
- 继承
- 多态

- 为什么 Java 只支持单继承？
- 多态的实现原理？

- 方法调用是动态绑定
- JVM 通过虚方法表查找

final 关键字（必问🔥）

- 修饰类 → 不能继承
- 修饰方法 → 不能重写
- 修饰变量 → 不能修改

- final 修饰对象，里面属性能改吗？

### 重载与重写

| 对比     | 重载     | 重写     |
| -------- | -------- | -------- |
| 发生位置 | 同类     | 子类     |
| 参数     | 必须不同 | 必须相同 |
| 返回值   | 可不同   | 必须兼容 |
| 访问权限 | 无限制   | 不能变小 |

- private 方法不能被重写
- static 是隐藏不是重写

### 构造器

- 构造器是否能重写？（不能）
- 父类构造什么时候执行？
- this() 和 super() 规则？

👉 super() 必须第一行。

- this 表示当前对象
- super 表示父类对象

常问：

- this() 和 super() 能否同时出现？

### Object方法

- 为什么必须同时重写？
- equals 默认比较什么？
- == 和 equals 区别？

👉 equals 默认比较地址。

- toString()
- equals()
- hashCode()
- wait()
- notify()
- clone()

- clone 是浅拷贝还是深拷贝？
- 为什么不推荐使用 clone？

### 抽象类与接口

Interface

| 对比   | 抽象类       | 接口             |
| ------ | ------------ | ---------------- |
| 继承   | 单继承       | 多实现           |
| 方法   | 可有普通方法 | Java8 可 default |
| 构造器 | 有           | 没有             |

面试爱问：

- 什么时候用抽象类？
- 什么时候用接口？

### 静态成员

- static 属于类
- 不能访问非静态成员

常问：

- 静态方法能否被重写？（不能）
- 类加载时机？

面试经典题：

> new 一个对象，执行顺序是什么？

顺序：

1. 父类静态代码块
2. 子类静态代码块
3. 父类实例代码块
4. 父类构造器
5. 子类实例代码块
6. 子类构造器

### 内部类

- 成员内部类
- 静态内部类
- 局部内部类
- 匿名内部类

追问：

- 为什么内部类可以访问外部类成员？

### 泛型

123

## 新特性

### Record

record 是在 Java 14 引入（preview），并在 Java 16 正式成为标准特性，其核心目标是为“纯数据载体类”提供更简洁、不可变的语法。

```java
public record Person(String name, int age) {}
```

编译器自动生成：

- private final fields
- constructor
- getter（name() 而不是 getName()）
- equals()
- hashCode()
- toString()

record 天生是不可变的，所有字段默认都是 private final，并且不会生成 setter 方法。

record 不能继承其他类（它默认继承 java.lang.Record），但可以实现接口。

record 可以自定义构造器，并且可以定义方法：

```java
public record Person(String name, int age) {
    public Person {
        if (age < 0) {
            throw new IllegalArgumentException("Age cannot be negative");
        }
    }
    public boolean isAdult() {
        return age >= 18;
    }
}
```

### Sealed Class

sealed class 是 Java 17 引入的特性，用于限制类的继承范围，只允许指定的子类进行扩展，从而实现受控继承。在Java 17之前，如果你想限制继承，只能用 final，但 final 会完全禁止继承，没办法“只允许特定子类”。

```java
// 只有 Circle 和 Rectangle 可以继承 Shape
public sealed class Shape
    permits Circle, Rectangle {
}

// 被允许继承的类，必须声明为：final/sealed/non-sealed
public final class Circle extends Shape {
}
```

不仅 class 可以 sealed，interface 也可以：

```java
public sealed interface Result
    permits Success, Failure {
}
```
