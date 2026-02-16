---
short_title: Concurrency
---

# 并发编程

# 🔥 第一部分：必问基础（不会基本挂）

---

## 1️⃣ 线程创建方式

### 常见问法：

- 创建线程有哪几种方式？
- Runnable 和 Callable 区别？
- Future 和 FutureTask 有什么区别？

### 标准答案：

1. 继承 Thread
2. 实现 Runnable
3. 实现 Callable + Future
4. 线程池（最推荐）

---

## 2️⃣ 线程生命周期（必问）

### 常见问法：

- 线程有哪些状态？
- BLOCKED 和 WAITING 区别？
- sleep 和 wait 区别？

### 线程状态：

- NEW
- RUNNABLE
- BLOCKED
- WAITING
- TIMED_WAITING
- TERMINATED

👉 sleep 不释放锁
👉 wait 释放锁

---

## 3️⃣ synchronized（超级高频🔥）

### 面试必问：

- synchronized 底层实现？
- 锁升级过程？
- 偏向锁 / 轻量级锁 / 重量级锁？
- 锁是公平的吗？

### 本质考察：

- 对象头 Mark Word
- Monitor
- CAS

---

## 4️⃣ volatile（必问🔥）

### 高频问题：

- volatile 解决什么问题？
- 能保证原子性吗？
- 底层怎么实现可见性？

👉 解决：

- 可见性
- 有序性
- 不能保证原子性

---

## 5️⃣ happens-before 原则（核心）

很多人背了却不懂。

常考：

- 为什么 volatile 能保证可见性？
- 指令重排序是什么？
- 双重检查锁为什么要 volatile？

---

# 🔥 第二部分：JUC（Java 并发包）—— 高频

来自：

Java Concurrency Utilities

---

## 6️⃣ 线程池（超级高频🔥🔥）

### 高频问题：

- 线程池核心参数？
- 拒绝策略有哪些？
- 线程池执行流程？
- 为什么不建议用 Executors？

核心类：

ThreadPoolExecutor

7 个核心参数必须会讲：

```java
corePoolSize
maximumPoolSize
keepAliveTime
workQueue
threadFactory
handler
```

---

## 7️⃣ 常见并发工具类

| 类             | 高频程度 |
| -------------- | -------- |
| CountDownLatch | 🔥🔥     |
| CyclicBarrier  | 🔥       |
| Semaphore      | 🔥       |
| ReentrantLock  | 🔥🔥     |
| ReadWriteLock  | 🔥       |
| AtomicInteger  | 🔥🔥     |

常问：

- ReentrantLock 和 synchronized 区别？
- 公平锁 vs 非公平锁？
- CAS 原理？

---

## 8️⃣ CAS（必问🔥）

### 常问：

- 什么是 CAS？
- ABA 问题？
- 自旋的优缺点？

---

# 🔥 第三部分：高阶原理（3年以上常问）

---

## 9️⃣ AQS（高阶必问🔥）

来自：

AbstractQueuedSynchronizer

问法：

- AQS 是什么？
- ReentrantLock 是如何基于 AQS 实现的？
- 同步队列是什么结构？

👉 本质：CLH 队列 + state + CAS

---

## 🔟 ConcurrentHashMap（必问🔥）

常问：

- JDK7 和 JDK8 区别？
- 扩容机制？
- 为什么线程安全？

---

## 11️⃣ 线程安全问题

- 死锁怎么产生？
- 如何排查死锁？
- 线程安全三要素？

---

# ⭐ 第四部分：Java 21 新并发（加分项）

来自：

Java 21

---

## 虚拟线程（Project Loom）

Project Loom

常问：

- 和传统线程区别？
- 为什么能创建百万级线程？
- 适用场景？

👉 IO 密集型提升明显。

---

# 🎯 面试真实高频排序（重要）

1. 线程池（必会）
2. synchronized + 锁升级
3. volatile
4. ReentrantLock
5. CAS
6. ConcurrentHashMap
7. AQS
8. CountDownLatch
9. 虚拟线程（了解）

---

# 🧠 面试官真正想看什么？

不是 API，而是：

| 能力     | 体现点                   |
| -------- | ------------------------ |
| 并发理解 | 可见性 / 原子性 / 有序性 |
| 原理深度 | CAS / AQS                |
| 实战经验 | 线程池参数如何调优       |
| 问题排查 | 死锁分析                 |

---

# 💡 如果你准备中级岗位（1-3年）

重点搞定：

- 线程池
- synchronized
- volatile
- CAS
- ConcurrentHashMap

就已经很稳。

---

如果你愿意，我可以：

- 🔥 给你整理一套“多线程面试 30 道高频真题 + 标准回答”
- 🔥 帮你做一张“Java 并发知识结构图”
- 🔥 帮你判断你现在水平在哪个段位

你现在几年经验？准备冲什么级别？
