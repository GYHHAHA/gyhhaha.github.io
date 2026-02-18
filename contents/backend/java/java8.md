---
short_title: New Features
---

# Java 新特性

## Lambda

### 基本用法

Java 8 的 Lambda 表达式本质上是对“函数式编程思想”在 Java 中的实现，它用更简洁的语法来替代匿名内部类，用于快速实现函数式接口。Lambda 并不是随便使用的，它的前提是目标类型必须是函数式接口——即只包含一个抽象方法的接口（可以包含多个 default 或 static 方法），通常配合 \@FunctionalInterface 注解进行约束和校验。

在 Java 接口中，普通方法默认就是 public abstract，即使不显式声明，编译器也会自动补全访问修饰符和抽象关键字。因此，在接口里写 void run(); 本质上等价于 public abstract void run();。只有当方法使用了 default、static（以及 Java 9 之后的 private）修饰时，才不是抽象方法。

常见的函数式接口：

| 接口            | 抽象方法                  | 作用说明             | 使用举例                                               |
| --------------- | ------------------------- | -------------------- | ------------------------------------------------------ |
| `Runnable`      | `void run()`              | 无参数、无返回值     | `new Thread(() -> System.out.println("run")).start();` |
| `Callable<V>`   | `V call()`                | 无参数、有返回值     | `ExecutorService.submit(() -> "result");`              |
| `Comparator<T>` | `int compare(T o1, T o2)` | 比较两个对象         | `list.sort((a, b) -> a - b);`                          |
| `Consumer<T>`   | `void accept(T t)`        | 有参数、无返回值     | `list.forEach(x -> System.out.println(x));`            |
| `Supplier<T>`   | `T get()`                 | 无参数、有返回值     | `Supplier<String> s = () -> "hello";`                  |
| `Function<T,R>` | `R apply(T t)`            | 有参数、有返回值     | `Function<Integer, Integer> f = x -> x * 2;`           |
| `Predicate<T>`  | `boolean test(T t)`       | 有参数、返回 boolean | `list.stream().filter(x -> x > 10);`                   |

Lambda 语法简化规则中，可以省略：

- 参数类型（编译器可推断）
- 单个参数的小括号
- 单行表达式的 return 和 {}

```java
(x, y) -> x + y
x -> x * 2
() -> System.out.println("hello")
```

Lambda 与匿名内部类的区别包括：

| 对比点              | Lambda          | 匿名内部类      |
| ------------------- | --------------- | --------------- |
| 作用对象            | 只能函数式接口  | 任意接口/抽象类 |
| this 指向           | 外部类          | 匿名类自身      |
| 是否生成 class 文件 | 不生成独立class | 会生成          |

以 Runnable 示例对比：

```java
// 写法冗长，明确创建了一个匿名内部类
// 编译后会生成一个新的 .class 文件
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello");
    }
}).start();

// 极简写法，本质是 Runnable 的实现
// 底层通过 invokedynamic 动态生成
new Thread(() -> System.out.println("Hello")).start();
```

以 Comparator 示例对比：

```java
List<Integer> list = Arrays.asList(3, 1, 2);

// 匿名内部类
Collections.sort(list, new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        return o1 - o2;
    }
});

// Lambda
Collections.sort(list, (o1, o2) -> o1 - o2);
```

Lambda 表达式可以访问外部作用域的局部变量、成员变量和静态变量，但其中局部变量必须是 final 或 effectively final。所谓 effectively final，是指变量虽然没有显式声明为 final，但在定义后其值没有被再次修改。之所以要求这样，是因为局部变量存储在栈内存中，而 Lambda 可能会在方法执行结束后才被调用（例如在线程或延迟执行场景中），此时原方法的栈帧已经销毁。为了保证变量在生命周期和并发环境下的安全性，Java 会在底层将这些变量“捕获”为一个不可变的副本，从而避免数据不一致和线程安全问题。因此，本质原因是变量生命周期与线程安全控制。

### 方法引用

1. 静态方法引用

写法： `类名::静态方法名`。用途： 当 Lambda 表达式中只是调用某个类的静态方法时，可以使用静态方法引用进行简化。

```java
// 普通 Lambda 写法
Function<String, Integer> f = s -> Integer.parseInt(s);

// 方法引用写法
Function<String, Integer> f = Integer::parseInt;
```

2.  实例方法引用（类名::实例方法）

写法： `类名::实例方法名`。用途： 当 Lambda 表达式的第一个参数作为调用对象去调用实例方法时，可以使用这种方式。第一个参数会被当作方法的调用者。

```java
// 普通 Lambda 写法
BiFunction<String, String, Boolean> f =
        (a, b) -> a.equals(b);

// 方法引用写法
BiFunction<String, String, Boolean> f =
        String::equals;
```

3.  对象方法引用（对象::实例方法）

写法： `对象::实例方法名`。用途： 当已经存在一个对象，Lambda 表达式只是调用该对象的实例方法时，可以使用对象方法引用。

```java
// 普通 Lambda 写法
PrintStream ps = System.out;
Consumer<String> c = s -> ps.println(s);

// 方法引用写法
PrintStream ps = System.out;
Consumer<String> c = ps::println;
```

4. 构造方法引用

写法： `类名::new`。用途： 当 Lambda 表达式只是调用构造方法创建对象时，可以使用构造方法引用。参数会自动匹配对应的构造器。

```java
// 普通 Lambda 写法
Function<String, User> f = name -> new User(name);

// 方法引用写法
Function<String, User> f = User::new;
```

## Stream

Stream 是一种数据处理管道，它本身不是数据结构，也不会存储数据，而是基于已有的数据源（如集合、数组）进行操作；在处理过程中不会改变原始数据，而是通过一系列支持链式调用的中间操作（如 filter、map 等）构建出一条操作流水线，并且这些操作采用惰性求值机制，只有在执行终止操作（如 collect、forEach、count 等）时，整个处理流程才会真正被触发执行。

Stream 的创建方式有：

| 分类     | 方法                      | 示例                 | 说明             |
| -------- | ------------------------- | -------------------- | ---------------- |
| 集合     | `collection.stream()`     | `list.stream()`      | 最常用方式       |
| 数组     | `Arrays.stream()`         | `Arrays.stream(arr)` | 支持基本类型数组 |
| 直接创建 | `Stream.of()`             | `Stream.of("a","b")` | 快速创建         |
| 整数区间 | `IntStream.range()`       | `range(1,5)`         | 左闭右开         |
|          | `IntStream.rangeClosed()` | `rangeClosed(1,5)`   | 闭区间           |

Stream 与集合的区别是：

| 对比点           | Collections              | Streams                    |
| ---------------- | ------------------------ | -------------------------- |
| 本质             | 数据结构                 | 数据处理管道               |
| 是否存储数据     | ✅ 存储数据              | ❌ 不存储数据              |
| 是否修改原数据   | ✅ 可以修改              | ❌ 不修改（函数式）        |
| 遍历方式         | 外部迭代（for/iterator） | 内部迭代                   |
| 编程风格         | 命令式                   | 函数式                     |
| 是否支持链式调用 | ❌ 基本不支持            | ✅ 支持                    |
| 是否支持并行     | ❌ 需要手动线程          | ✅ parallelStream()        |
| 执行方式         | 立即执行                 | 惰性执行（终止操作才触发） |
| 适用场景         | 数据存储、CRUD           | 数据筛选、转换、聚合       |

### 中间操作

常见中间操作有：

| 分类            | 方法                 | 作用         | 是否有状态 | 是否改变元素 | 典型场景           |
| --------------- | -------------------- | ------------ | ---------- | ------------ | ------------------ |
| 过滤类          | `filter()`           | 条件筛选元素 | ❌ 无状态  | ❌ 不改变    | 条件查询           |
| 映射类          | `map()`              | 一对一转换   | ❌ 无状态  | ✅ 改变      | 字段提取、类型转换 |
|                 | `flatMap()`          | 一对多打平   | ❌ 无状态  | ✅ 改变      | 嵌套集合展开       |
| 截断类（Slice） | `limit()`            | 取前 n 个    | ✅ 有状态  | ❌ 不改变    | 分页               |
|                 | `skip()`             | 跳过前 n 个  | ✅ 有状态  | ❌ 不改变    | 分页               |
| 去重类          | `distinct()`         | 去除重复     | ✅ 有状态  | ❌ 不改变    | 去重               |
| 排序类          | `sorted()`           | 自然排序     | ✅ 有状态  | ❌ 不改变    | 排序               |
|                 | `sorted(Comparator)` | 自定义排序   | ✅ 有状态  | ❌ 不改变    | 多字段排序         |
| 调试类          | `peek()`             | 查看流元素   | ❌ 无状态  | ❌ 不改变    | 调试               |

```java
import java.util.*;
import java.util.stream.*;

public class StreamDemo {
    public static void main(String[] args) {
        List<List<String>> data = Arrays.asList(
                Arrays.asList("apple", "banana", "apple"),
                Arrays.asList("orange", "banana"),
                Arrays.asList("pear", "kiwi")
        );
        List<String> result = data.stream()
                // 1️⃣ flatMap：打平
                // [apple, banana, apple, orange, banana, pear, kiwi]
                .flatMap(Collection::stream)
                // 2️⃣ filter：长度 > 5
                // [banana, orange, banana]
                .filter(s -> s.length() > 5)
                // 3️⃣ map：转大写
                // [BANANA, ORANGE, BANANA]
                .map(String::toUpperCase)
                // 4️⃣ distinct：去重
                // [BANANA, ORANGE]
                .distinct()
                // 5️⃣ sorted：排序
                // [BANANA, ORANGE]
                .sorted()
                // 6️⃣ skip(1)：跳过第一个
                // [ORANGE]
                .skip(1)
                // 7️⃣ limit(2)：取前 2 个
                // [ORANGE]
                .limit(2)
                // 8️⃣ peek：查看
                .peek(s -> System.out.println("转大写后: " + s))
                // 终止操作
                .collect(Collectors.toList());
        // 最终结果
        // [ORANGE]
        System.out.println(result);
    }
}
```

### 终止操作

常见的终止操作有：

| 分类     | 方法                | 返回值                | 作用说明                       |
| -------- | ------------------- | --------------------- | ------------------------------ |
| 遍历执行 | `forEach`           | void                  | 遍历元素执行操作               |
| 计数统计 | `count`             | long                  | 统计元素个数                   |
| 匹配判断 | `anyMatch`          | boolean               | 是否存在满足条件的元素         |
|          | `allMatch`          | boolean               | 是否全部满足条件               |
|          | `noneMatch`         | boolean               | 是否全部不满足条件             |
| 查找操作 | `findFirst`         | Optional              | 返回第一个元素                 |
|          | `findAny`           | Optional              | 返回任意元素（并行流更高效）   |
| 归约操作 | `reduce`            | Optional / T          | 聚合成一个值（求和、拼接等）   |
| 收集操作 | `collect`           | R                     | 收集结果                       |
|          | `groupingBy`        | Map<K, List<T>>       | 按字段分组                     |
|          | `partitioningBy`    | Map<Boolean, List<T>> | 按条件分区                     |
|          | `collectingAndThen` | R                     | 收集后再转换                   |
|          | `joining`           | String                | 拼接字符串                     |
|          | `summarizing`       | 统计对象              | 汇总统计（最大、最小、平均等） |

```java
import java.util.*;
import java.util.stream.*;
import static java.util.stream.Collectors.*;

public class StreamTerminalDemo {

    public static void main(String[] args) {

        List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6);

        // 1️⃣ forEach（遍历）
        list.stream()
                .forEach(n -> System.out.print(n + " "));
        // 输出: 1 2 3 4 5 6

        System.out.println();

        // 2️⃣ count（计数）
        long count = list.stream().count();
        System.out.println("count = " + count); // 6

        // 3️⃣ anyMatch / allMatch / noneMatch（匹配判断）
        boolean anyGreaterThan5 = list.stream().anyMatch(n -> n > 5);
        boolean allGreaterThan0 = list.stream().allMatch(n -> n > 0);
        boolean noneNegative = list.stream().noneMatch(n -> n < 0);

        System.out.println(anyGreaterThan5); // true
        System.out.println(allGreaterThan0); // true
        System.out.println(noneNegative);    // true

        // 4️⃣ findFirst / findAny（查找）
        Optional<Integer> first = list.stream().findFirst();
        Optional<Integer> any = list.parallelStream().findAny();

        System.out.println(first.get()); // 1
        System.out.println(any.get());   // 任意一个元素

        // 5️⃣ reduce（归约）
        int sum = list.stream().reduce(0, Integer::sum);
        System.out.println("sum = " + sum); // 21

        // 6️⃣ collect + groupingBy（分组：按奇偶）
        Map<String, List<Integer>> group =
                list.stream().collect(groupingBy(n -> n % 2 == 0 ? "even" : "odd"));
        System.out.println(group);
        // {even=[2,4,6], odd=[1,3,5]}

        // 7️⃣ partitioningBy（分区：大于3）
        Map<Boolean, List<Integer>> partition =
                list.stream().collect(partitioningBy(n -> n > 3));
        System.out.println(partition);
        // {false=[1,2,3], true=[4,5,6]}

        // 8️⃣ joining（拼接）
        String joined = list.stream()
                .map(String::valueOf)
                .collect(joining(","));
        System.out.println(joined); // 1,2,3,4,5,6

        // 9️⃣ summarizing（统计汇总）
        IntSummaryStatistics stats =
                list.stream().collect(summarizingInt(Integer::intValue));
        System.out.println(stats);
        // count=6, sum=21, min=1, average=3.5, max=6

        // 🔟 collectingAndThen（收集后再处理）
        Integer max = list.stream()
                .collect(collectingAndThen(maxBy(Integer::compareTo), Optional::get));
        System.out.println("max = " + max); // 6
    }
}
```

### 其他

iterate 用于按照一定规则生成递增（或递推）序列，例如基于前一个元素计算下一个元素；generate 用于按照给定规则不断生成元素（如随机数或固定值）；这两种方式默认都会创建无限流，如果不加限制会一直执行下去，因此在实际使用时通常必须配合 limit() 进行截断，控制生成元素的数量。

```java
import java.util.stream.*;

public class InfiniteStreamDemo {
    public static void main(String[] args) {

        // 1️⃣ iterate：递增序列
        Stream.iterate(0, n -> n + 1)
                .limit(5)
                .forEach(System.out::println);
        // 输出：0 1 2 3 4

        // 2️⃣ generate：随机数
        Stream.generate(Math::random)
                .limit(3)
                .forEach(System.out::println);
        // 输出：3个随机小数
    }
}
```

Optional 是为了解决“可能为空”的返回值问题，Stream 的 findFirst / findAny / max / min 等终止操作经常返回 Optional，你需要会用 ifPresent / orElse / orElseGet / orElseThrow 来安全处理结果，避免空指针。

```java
import java.util.*;

public class OptionalDemo {
    static class User {
        String name;
        User(String name) { this.name = name; }
        public String toString() { return "User{name='" + name + "'}"; }
    }

    public static void main(String[] args) {
        List<User> list = Arrays.asList(new User("Tom"), new User("Jerry"));

        Optional<User> opt = list.stream()
                .filter(u -> u.name.equals("Alice"))
                .findFirst(); // 可能找不到 -> Optional.empty()

        opt.ifPresent(u -> System.out.println("找到了: " + u));

        User user = opt.orElse(new User("Default")); // 找不到就给默认值
        System.out.println("最终使用: " + user);
    }
}
```

并行流 parallelStream() 底层使用 ForkJoinPool 把任务拆分到多个线程并行执行，但要注意：并行并不总更快（拆分/合并有开销），而且对共享可变状态非常敏感，forEach 输出顺序也不保证；因此遇到需要严格顺序、需要访问共享资源（如往同一个 ArrayList add）、IO 密集型、数据量很小等场景通常不建议用并行流。

```java
import java.util.*;
import java.util.stream.*;

public class ParallelStreamDemo {
    public static void main(String[] args) {

        // ✅ 推荐：无共享可变状态 + 计算密集
        int sum = IntStream.rangeClosed(1, 1_000_000)
                .parallel()                  // 并行执行（ForkJoinPool）
                .map(x -> x * x)             // 计算密集
                .sum();                      // 并行安全的归约
        System.out.println("sum=" + sum);

        // ❌ 不推荐：共享可变集合（线程安全问题）
        List<Integer> bad = new ArrayList<>();
        IntStream.rangeClosed(1, 10_000)
                .parallel()
                .forEach(bad::add);          // 可能丢数据/越界/结果不稳定
        System.out.println("bad size=" + bad.size());
    }
}
```

## Optional

Optional 是 Java 8 引入的用于显式处理空值的容器类，核心用法围绕创建（of/ofNullable）、安全取值（orElse/orElseGet）和函数式转换（map/flatMap），Optional 主要用于方法返回值。

1. 创建 Optional

| 方法                         | 作用              | 说明              |
| ---------------------------- | ----------------- | ----------------- |
| `Optional.of(value)`         | 创建非空 Optional | value 不能为 null |
| `Optional.ofNullable(value)` | 创建可空 Optional | 推荐使用          |
| `Optional.empty()`           | 创建空 Optional   | 无值              |

2. 判断 Optional 是否为空

| 方法                   | 作用     |
| ---------------------- | -------- |
| `isPresent()`          | 是否有值 |
| `isEmpty()` (Java 11+) | 是否为空 |

3. 取值

| 方法                  | 作用           | 是否推荐  |
| --------------------- | -------------- | --------- |
| `get()`               | 直接取值       | ❌ 不推荐 |
| `orElse(default)`     | 无值返回默认值 | ✅        |
| `orElseGet(supplier)` | 懒加载默认值   | ✅        |
| `orElseThrow()`       | 无值抛异常     | ✅        |

orElse和orElseGet的区别：

```java
optional.orElse(expensiveMethod());     // 总会执行
optional.orElseGet(() -> expensive());  // 只有为空才执行
```

4. 函数式处理

| 方法          | 作用              |
| ------------- | ----------------- |
| `ifPresent()` | 有值时执行        |
| `map()`       | 转换值            |
| `flatMap()`   | 避免嵌套 Optional |
| `filter()`    | 条件过滤          |

```java
optional
    .filter(s -> s.length() > 3)
    .map(String::toUpperCase)
    .ifPresent(System.out::println);
```

## DateTime

所有 java.time 类都是不可变的，因为不可变，所以线程安全。

核心类包括：

| 类                  | 作用                 | 常见用途       |
| ------------------- | -------------------- | -------------- |
| `LocalDate`         | 只有日期（年-月-日） | 生日、账单日期 |
| `LocalTime`         | 只有时间（时-分-秒） | 营业时间       |
| `LocalDateTime`     | 日期 + 时间          | 业务时间       |
| `ZonedDateTime`     | 带时区时间           | 国际化系统     |
| `Instant`           | 时间戳（UTC）        | 存数据库       |
| `Duration`          | 时间间隔（秒/毫秒）  | 计算耗时       |
| `Period`            | 日期间隔（年/月/日） | 计算年龄       |
| `DateTimeFormatter` | 格式化工具           | 字符串转换     |

常用方法：

```java
// 1️⃣ 获取当前时间
LocalDate today = LocalDate.now();
LocalDateTime now = LocalDateTime.now();
Instant instantNow = Instant.now();

// 2️⃣ 创建指定日期时间
LocalDate date = LocalDate.of(2025, 3, 1);
LocalDateTime dateTime = LocalDateTime.of(2025, 3, 1, 10, 30);

// 3️⃣ 时间加减（不可变，会返回新对象）
LocalDate nextWeek = today.plusDays(7);
LocalDate lastMonth = today.minusMonths(1);

// 4️⃣ 时间比较
boolean isBefore = date.isBefore(today);
boolean isAfter = date.isAfter(today);

// 5️⃣ 计算时间差（Duration 适用于时间差）
Duration duration = Duration.between(
        LocalDateTime.of(2025, 3, 1, 10, 0),
        LocalDateTime.of(2025, 3, 1, 12, 30)
);
long minutes = duration.toMinutes();

// 6️⃣ 格式化和解析
DateTimeFormatter formatter =
        DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String formatted = now.format(formatter);
LocalDateTime parsed =
        LocalDateTime.parse("2025-03-01 10:30:00", formatter);

// 7️⃣ 时间戳转换（Instant ↔ LocalDateTime）
LocalDateTime fromInstant =
        LocalDateTime.ofInstant(instantNow, ZoneId.systemDefault());
```

一个东八区时间转西五区的例子：

```java
import java.time.*;
import java.time.format.DateTimeFormatter;

LocalDateTime dbTime = LocalDateTime.parse("2026-02-16T10:00:00");

// 1) 解释为东八区（Asia/Shanghai 或你实际用的东8区 ZoneId）
ZonedDateTime chinaTime = dbTime.atZone(ZoneId.of("Asia/Shanghai"));

// 2) 转成西五区（两种写法：固定 -05:00 或具体地区时区）
// 固定西5
ZonedDateTime westFixed = chinaTime.withZoneSameInstant(ZoneOffset.ofHours(-5));
// 纽约时区（有夏令时）
ZonedDateTime newYork = chinaTime.withZoneSameInstant(ZoneId.of("America/New_York"));

// 3) 输出给前端（按你需要的格式）
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String out1 = westFixed.format(fmt);
String out2 = newYork.format(fmt);
```
