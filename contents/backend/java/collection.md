---
short_title: Collection
---

- List.of()

# 集合框架

Java 集合框架是后端面试**绝对核心高频模块**（和多线程一个级别）。
面试官一般会围绕：

> ✅ 数据结构原理
> ✅ 源码理解
> ✅ 使用场景
> ✅ 性能对比
> ✅ 并发问题

我给你按“🔥必问 / ⭐常问 / 💡进阶”分层梳理一遍。

---

# 一、整体结构（基础必会🔥）

Java 集合框架核心分两大体系：

来自：

Java Collections Framework

---

## 1️⃣ Collection 体系（单列）

- List
- Set
- Queue

---

## 2️⃣ Map 体系（键值对）

- HashMap
- LinkedHashMap
- TreeMap
- ConcurrentHashMap

---

# 二、List 系列（高频🔥🔥🔥）

---

## 1️⃣ ArrayList（必问）

![Image](https://www.c-sharpcorner.com/UploadFile/230635/an-extensive-examination-of-arraylist-in-C-Sharp/Images/arraylist-structure.png)

![Image](https://miro.medium.com/v2/resize%3Afit%3A1400/1%2AqAy9S8kwumSloS8xADnsDg.png)

![Image](https://i.sstatic.net/Ly4Fp.jpg)

![Image](https://i.sstatic.net/SpCcj.png)

来自：

ArrayList

### 高频问题：

- 底层数据结构？
- 扩容机制？
- 为什么是 1.5 倍扩容？
- 插入删除时间复杂度？
- 是否线程安全？

### 核心知识点：

- 底层是数组
- 默认容量 10
- 扩容 newCapacity = old + old >> 1
- 查询快，插入慢（中间插入 O(n)）

---

## 2️⃣ LinkedList（必问）

![Image](https://www.alphacodingskills.com/imgfiles/doubly-linked-list.PNG)

![Image](https://www.alphacodingskills.com/imgfiles/linked-list.PNG)

![Image](https://favtutor.com/resources/images/uploads/figure_5-01_%281%29_%281%29.jpg)

![Image](https://deen3evddmddt.cloudfront.net/uploads/content-images/singly-linked-list-operations-in-data-structure.webp)

来自：

LinkedList

### 高频问题：

- 底层结构？
- 和 ArrayList 区别？
- 适用场景？

👉 双向链表
👉 插入删除快，查询慢

---

## 3️⃣ ArrayList vs LinkedList（经典对比🔥）

| 对比点 | ArrayList | LinkedList |
| ------ | --------- | ---------- |
| 结构   | 数组      | 双向链表   |
| 查询   | 快        | 慢         |
| 插入   | 慢        | 快         |
| 内存   | 连续      | 离散       |

---

# 三、Set 系列（高频🔥🔥）

---

## 1️⃣ HashSet（必问）

来自：

HashSet

### 高频问题：

- 底层是什么？
- 为什么元素不重复？
- equals 和 hashCode 关系？

👉 底层就是 HashMap

---

## 2️⃣ TreeSet（常问）

来自：

TreeSet

问：

- 底层数据结构？
- 如何排序？

👉 底层是红黑树（基于 TreeMap）

---

# 四、Map 系列（超级高频🔥🔥🔥）

---

## 1️⃣ HashMap（面试王者🔥🔥🔥）

![Image](https://miro.medium.com/1%2AjPM0jwnyL_przj93M_u0bQ.png)

![Image](https://miro.medium.com/v2/resize%3Afit%3A1400/1%2AJHi4-murLVxHOPAcBOX7fA.png)

![Image](https://miro.medium.com/1%2Aw1mRVHC1hNc2ywDoYibkiA.jpeg)

![Image](https://miro.medium.com/v2/resize%3Afit%3A1200/0%2A8R3e49BbAuMEJhrx.jpg)

来自：

HashMap

### 高频问题：

- 底层结构？
- JDK7 和 JDK8 区别？
- 为什么要红黑树？
- 扩容机制？
- 为什么线程不安全？
- 1.8 为什么优化了死循环问题？

### 核心知识点：

- 数组 + 链表 + 红黑树
- 默认容量 16
- 负载因子 0.75
- 链表长度 ≥ 8 转红黑树

---

## 2️⃣ LinkedHashMap（常问）

来自：

LinkedHashMap

问：

- 和 HashMap 区别？
- LRU 如何实现？

👉 多了双向链表维护顺序。

---

## 3️⃣ TreeMap（常问）

来自：

TreeMap

问：

- 底层结构？
- 和 HashMap 区别？

👉 红黑树，自动排序。

---

## 4️⃣ ConcurrentHashMap（高频🔥）

来自：

ConcurrentHashMap

问：

- JDK7 vs JDK8 区别？
- 为什么线程安全？
- CAS + synchronized 怎么结合？

---

# 五、源码常考点（中高级必问🔥）

---

## 1️⃣ HashMap 扩容流程

- 什么时候扩容？
- 扩容是否重新 hash？
- 为什么 2 的幂次？

---

## 2️⃣ fail-fast 机制

问：

- 什么是 fail-fast？
- 为什么会 ConcurrentModificationException？

👉 modCount 机制。

---

## 3️⃣ equals 和 hashCode

经典问题：

> 如果只重写 equals 不重写 hashCode 会怎样？

---

# 六、面试真实高频排序（经验排序）

1. 🔥 HashMap
2. 🔥 ArrayList
3. 🔥 ConcurrentHashMap
4. 🔥 equals/hashCode
5. 🔥 扩容机制
6. ⭐ LinkedHashMap（LRU）
7. ⭐ TreeMap
8. ⭐ fail-fast

---

# 七、面试官真正想看什么？

| 考察点          | 真正考你     |
| --------------- | ------------ |
| HashMap         | 数据结构理解 |
| 扩容            | 性能思维     |
| 并发Map         | 多线程能力   |
| equals/hashCode | 基础扎实度   |
| LRU             | 设计能力     |

---

# 🎯 如果你是 1-3 年经验

必须精通：

- HashMap 原理
- ArrayList 扩容
- ConcurrentHashMap
- equals/hashCode
- fail-fast

---

如果你愿意，我可以给你：

- 🔥 集合框架 40 道高频面试题 + 标准回答
- 🔥 HashMap 源码深度拆解
- 🔥 一张集合结构脑图（面试复习版）

你现在几年经验？我可以帮你划一个“面试必会范围”。
