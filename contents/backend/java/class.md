---
short_title: Class & Object
---

# 面向对象

## 基本概念

### 类与修饰符

一个 `.java` 文件中，最多只能有一个 public 顶级类，文件名必须和这个 public 类名一致。Java 中不能有独立函数，函数必须属于类

Java 通过四种修饰符来控制类、属性、方法的可见性。

| 修饰符    | 本类 | 同一个包 | 子类（不同包） | 任意位置（不同包） | 备注                                   |
| --------- | ---- | -------- | -------------- | ------------------ | -------------------------------------- |
| private   | ✅   | ❌       | ❌             | ❌                 | 封装的核心。仅限类内部使用。           |
| default   | ✅   | ✅       | ❌             | ❌                 | 不写任何关键字时的默认状态（包权限）。 |
| protected | ✅   | ✅       | ✅             | ❌                 | 继承的核心。专门给子类留的后门。       |
| public    | ✅   | ✅       | ✅             | ✅                 | 只要能导入这个包，就能访问。           |

### 三大特性

1. 封装（Encapsulation）的核心思想是隐藏内部实现，只对外暴露必要接口。

```java
class User {
    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

2. 继承（Inheritance）的核心思想是子类复用父类代码。

```java
class Animal {
    public void eat() {
        System.out.println("吃东西");
    }
}

class Dog extends Animal {
}
```

Java 只支持单继承，是为了避免多继承带来的“菱形继承”二义性问题，防止子类在继承多个父类时对同一方法或属性产生调用冲突，从而保证继承结构清晰、逻辑确定、实现简单。同时，Java 通过允许接口多实现来弥补单继承在扩展性上的不足，实现“类单继承 + 接口多实现”的设计平衡。

final 关键词可以修饰类、方法、变量，用于限定修饰对象的使用范围：

| 修饰对象         | 示例                         | 作用             | 典型场景               | 底层影响                           |
| ---------------- | ---------------------------- | ---------------- | ---------------------- | ---------------------------------- |
| 类               | `final class A {}`           | 不能被继承       | 工具类、安全类         | 类不会有子类，结构固定             |
| 方法             | `public final void test(){}` | 不能被重写       | 不希望子类修改核心逻辑 | 可被 JVM 内联优化（静态绑定）      |
| 变量（基本类型） | `final int a = 10;`          | 只能赋值一次     | 常量定义               | 值不可改变                         |
| 变量（引用类型） | `final User u = new User();` | 引用地址不能改变 | 防止对象被重新指向     | 不能指向新对象，但对象内部属性可变 |

3. 多态（Polymorphism）的核心思想是父类引用指向子类对象，方法调用是动态绑定，即编译时不知道调用哪个方法，运行时才确定调用子类的方法。多态的三个前提是：有继承、有方法重写、父类引用指向子类对象。

在 Java 中，多态的底层依赖于 虚方法表（vtable）机制。JVM 会为每个类在方法区维护一张虚方法表，表中记录该类可被重写的实例方法地址。当通过父类引用调用方法时，编译阶段只确定“方法符号引用”，真正执行哪个方法要等到运行时：JVM 先根据对象找到其真实类型（如 Dog），再到该类的虚方法表中查找对应的方法入口，最终调用子类重写后的实现。这就是动态绑定。与之对比，像 static、private、final 方法在编译期就能确定调用版本，属于静态绑定；而普通成员方法默认采用运行期确定的动态绑定机制。

```java
Animal a = new Dog();
a.eat(); // 实际调用 Dog 的 eat()
```

### 重载与重写

- 重载（Overload）：在同一个类中方法名相同但参数列表不同，与返回值无关，本质是方法签名不同，属于编译期确定的静态多态。
- 重写（Override）：子类在继承关系中对父类方法进行重新实现，要求方法名和参数完全相同、返回值兼容、权限不能缩小、异常不能扩大，并在运行期通过动态绑定实现多态。

| 对比项       | 重载（Overload）               | 重写（Override）   |
| ------------ | ------------------------------ | ------------------ |
| 发生位置     | 同一个类中                     | 子类中             |
| 方法名       | 必须相同                       | 必须相同           |
| 参数列表     | 必须不同（类型 / 个数 / 顺序） | 必须相同           |
| 返回值       | 可以不同                       | 必须相同或协变返回 |
| 访问权限     | 无限制                         | 不能变小           |
| 发生阶段     | 编译期（静态绑定）             | 运行期（动态绑定） |
| 是否要求继承 | 不需要                         | 必须有继承关系     |

private 方法不能被重写，因为它不参与继承，子类中即使定义了同名同参的方法，也只是一个新的方法，与父类方法没有重写关系。

```java
class A {
    private void test() {}
}

class B extends A {
    public void test() {} // 不是重写，是新方法
}
```

static 方法不能被重写，只能被隐藏，因为它属于类而不是对象，调用在编译期就已确定，不参与运行期的动态绑定。

```java
class A {
    static void test() {
        System.out.println("A");
    }
}

class B extends A {
    static void test() {
        System.out.println("B");
    }
}

public class Main {
    public static void main(String[] args) {
        A a = new B();
        a.test();   // 输出：A
    }
}
```

### 构造器

构造器是与类名相同、没有返回值的方法，在创建对象时自动调用，可以重载但不能被继承或重写；构造器不能重写的原因是它不参与继承且子类不会继承父类构造器；当创建子类对象时，一定会先执行父类构造器，再执行子类构造器，保证对象从父到子完成初始化。

super() 用于调用父类构造器，如果不写，默认隐式调用 super()，this() 用于调用本类其他构造器，二者都必须写在构造器的第一行；由于构造器的第一行只能有一条语句，因此 this() 和 super() 不能同时出现，只能二选一。

| 对比           | this         | super         |
| -------------- | ------------ | ------------- |
| 表示           | 当前对象     | 父类对象      |
| 访问成员       | 本类成员     | 父类成员      |
| 调用构造       | 调本类构造器 | 调父类构造器  |
| 是否必须第一行 | 仅 this() 时 | 仅 super() 时 |

构造器执行完整流程为：

- 先进入子类构造器
- 第一行调用 super()
- 进入父类构造器
- 父类构造执行完
- 返回子类构造器继续执行

### Object方法

常见的Object方法包括：

| 方法       | 作用                                            |
| ---------- | ----------------------------------------------- |
| toString() | 返回对象字符串表示（默认为类名@十六进制hash值） |
| equals()   | 比较对象是否相等                                |
| hashCode() | 返回对象哈希值                                  |
| wait()     | 当前线程进入等待                                |
| notify()   | 唤醒一个等待线程                                |
| clone()    | 复制对象                                        |

注意，必须同时重写 equals 和 hashCode。如果只重写 equals() 而不重写 hashCode()，就会破坏 Java 的约定：两个对象如果 equals 相等，它们的 hashCode 必须相等。因为在 HashMap、HashSet 等基于哈希的集合中，元素存取时会先比较 hashCode 决定桶位置，再用 equals 判断是否为同一个对象；如果两个逻辑上相等的对象 hashCode 不同，就会被分到不同位置，从而导致集合中出现重复元素或查找失败的问题。

### 抽象类与接口

什么时候用抽象类？当多个子类之间存在明显的继承关系，并且需要共享属性、通用方法或部分实现逻辑时，应使用抽象类，它既可以提供公共代码实现，又可以定义抽象方法强制子类实现，适合强调“is-a”关系和代码复用的场景。

什么时候用接口？当类之间不强调继承关系，而只是需要约定某种行为规范或扩展某种能力时，应使用接口，它更侧重定义标准和能力扩展，支持多实现，适合解耦设计和横向能力扩展的场景。

| 对比项   | 抽象类                   | 接口                                 |
| -------- | ------------------------ | ------------------------------------ |
| 继承方式 | 单继承                   | 多实现                               |
| 方法     | 可以有普通方法、抽象方法 | Java 8 之后可有 default、static 方法 |
| 成员变量 | 可以有普通成员变量       | 只能是 `public static final` 常量    |
| 构造器   | 有                       | 没有                                 |
| 设计目的 | 代码复用                 | 规范约束                             |

### 静态成员

static 属于类，而不是对象，不依赖对象，也不参与多态。随类加载而加载，只执行一次。静态方法中不能访问非静态成员（因为没有对象）。静态方法不能被重写，只能被隐藏（编译期绑定）。类在首次主动使用时才会被加载和初始化，例如创建对象、访问静态变量、调用静态方法、通过反射使用类或初始化子类时触发父类初始化，并且一个类在 JVM 生命周期中只会被加载和初始化一次。

静态代码块在类加载阶段执行且只执行一次，实例代码块在每次创建对象时都会执行，而构造器是在对象初始化过程中的最后一步执行。new 一个对象执行顺序为：

- 父类静态代码块
- 子类静态代码块
- 父类实例代码块
- 父类构造器
- 子类实例代码块
- 子类构造器

### 内部类

1. 成员内部类：定义在类中方法外，依赖外部类对象存在。由于是因为编译后内部类会隐式持有一个外部类对象引用，因此可以访问外部类的所有成员（包括 private）。

```java
class Outer {
    private int a = 10;

    class Inner {
        void test() {
            System.out.println(a); // 可以访问
        }
    }
}
```

2. 静态内部类：使用 static 修饰，不依赖外部类对象，只能访问外部类的静态成员。

```java
class Outer {
    static int a = 10;

    static class Inner {
        void test() {
            System.out.println(a);
        }
    }
}
```

3. 局部内部类：定义在方法内部，作用范围仅限当前方法，并且只能访问方法中有效 final 的变量。

```java
void method() {
    int x = 10; // 不能再修改
    class Inner {
        void test() {
            System.out.println(x);
        }
    }
}
```

4. 匿名内部类：没有类名，通常用于临时实现接口或继承类，多用于回调或事件处理场景。

```java
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.println("run");
    }
};
```

### 枚举

枚举是一种 特殊的类，用于表示一组固定常量，本质是一个继承自 java.lang.Enum 的类，用于定义固定常量集合，类型安全、可扩展、线程安全，常用于状态表示和单例模式实现。

| 特性             | 说明         |
| ---------------- | ------------ |
| 默认继承         | 继承 Enum    |
| 构造器           | 默认 private |
| 可以定义方法     | ✅           |
| 可以定义成员变量 | ✅           |
| 可以实现接口     | ✅           |
| 不能继承类       | ❌           |

```java
public enum Color {
    RED,
    GREEN,
    BLUE
}
```

枚举不仅可以定义常量，还可以像普通类一样定义成员变量、构造器和方法；其构造器默认是 private，用于在创建每个枚举常量时初始化属性，并且枚举常量会在类加载时一次性创建完成，因此枚举实例是固定且唯一的。

```java
enum Color {
    RED("红色"),
    GREEN("绿色"),
    BLUE("蓝色");

    private String desc;

    Color(String desc) {
        this.desc = desc;
    }

    public String getDesc() {
        return desc;
    }
}
```

枚举类自带一些常用方法，例如 values() 用于获取所有枚举值，valueOf(String) 根据名称获取对应枚举对象，name() 返回枚举常量名称，ordinal() 返回枚举在声明时的索引位置，这些方法方便对枚举进行遍历、查找和状态判断。

### 泛型

Java 的泛型采用的是类型擦除机制，也就是说泛型只在编译期存在，用于类型检查和保证类型安全，编译完成后会被擦除，运行时统一变成原始类型，例如 List<String> 和 List<Integer> 在运行时都会变成 List。正因为类型信息在运行时被擦除，所以泛型不能使用基本类型（如 List<int> 不合法，只能使用包装类型 List<Integer>，因为擦除后默认以 Object 作为上界，而基本类型不是对象），也不能在泛型中 new T()（因为运行时无法确定 T 的具体类型），同时也不能创建泛型数组或使用 instanceof 判断具体泛型类型，这些限制本质上都来源于类型擦除机制。

1. 泛型类

```java
class Box<T> {
    private T value;

    public void set(T value) {
        this.value = value;
    }

    public T get() {
        return value;
    }
}
```

2. 泛型方法

```java
public <T> T getFirst(T a) {
    return a;
}
```

3. 泛型接口

```java
public interface Box<T> {
    T get();
}
```

4. 泛型通配符

| 写法            | 名称       | 含义       | 典型场景             |
| --------------- | ---------- | ---------- | -------------------- |
| `<?>`           | 无界通配符 | 任意类型   | 不关心具体类型       |
| `<? extends T>` | 上界通配符 | T 及其子类 | 生产数据（Producer） |
| `<? super T>`   | 下界通配符 | T 及其父类 | 消费数据（Consumer） |

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
