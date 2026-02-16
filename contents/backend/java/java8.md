---
short_title: New Features
---

# Java 新特性

## Lambda

## Stream

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
