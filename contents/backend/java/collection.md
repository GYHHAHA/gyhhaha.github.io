---
short_title: Collection
---

# 集合框架

Java 集合框架整体可以分为两大体系：Collection 单列集合 和 Map 双列集合。

fail-fast
: 当集合在迭代过程中被其他线程或代码修改时，会立即抛出异常，而不是继续执行。Java 集合在使用迭代器（Iterator）遍历时，如果检测到集合结构被修改（如 add、remove），就会快速失败并抛出 ConcurrentModificationException。每次发生结构性修改（增删元素）时，都会让 modCount++。当创建迭代器时，会保存一份当前的 expectedModCount。在遍历过程中，迭代器会不断检查，一旦发现不一致，说明集合在遍历期间被修改，就会抛出异常。因此本质是通过 版本号对比机制 来检测非法修改。这种机制的目的不是保证线程安全，而是尽早发现并发修改问题，避免产生不可预期结果。

## Collection

Collection 用于存储单个元素对象，是最基础的集合体系。它下面主要分为三大类：

- List：有序、元素可重复，支持按索引访问，典型实现如 ArrayList、LinkedList。
- Set：无序（或有排序规则）、元素不可重复，底层通常依赖 Hash 或红黑树实现去重。
- Queue：队列结构，通常遵循 FIFO（先进先出）规则，适用于任务调度、缓存等场景。

### List

常见的List实现有ArrayList和LinkedList。

| 维度     | ArrayList             | LinkedList                    |
| -------- | --------------------- | ----------------------------- |
| 底层结构 | 动态数组（连续内存）  | 双向链表（非连续内存）        |
| 查询性能 | 快（O(1)）            | 慢（O(n)）                    |
| 插入删除 | 慢（需移动元素 O(n)） | 较快（修改指针，定位后 O(1)） |
| 内存占用 | 较小                  | 较大（额外前后指针）          |
| 适用场景 | 查询多、修改少        | 增删多、随机访问少            |

#### ArrayList

底层实现
: ArrayList 底层是 动态数组（Object[] elementData），通过下标进行元素访问，因此查询时间复杂度为 O(1)。它内部用 size 记录实际元素个数。需要注意的是，JDK8 中无参构造时并不会直接创建容量为 10 的数组，而是先创建一个空数组，在第一次添加元素时才初始化为默认容量 10。
: 当元素个数超过当前数组容量时会触发扩容。扩容的核心公式是：
newCapacity = oldCapacity + (oldCapacity >> 1)，也就是 1.5 倍扩容。例如 10 扩为 15，15 扩为 22。扩容本质是创建新数组并复制旧数据（System.arraycopy），时间复杂度为 O(n)。之所以采用 1.5 倍，是在扩容频率和内存利用率之间的折中：倍数过大浪费空间，倍数过小扩容过于频繁。

插入删除时间复杂度
: 由于底层是连续内存的数组结构，尾部添加元素在不扩容的情况下是 O(1)，均摊时间复杂度也是 O(1)。但如果在中间插入或删除元素，需要将后续元素整体移动，因此时间复杂度为 O(n)。因此 ArrayList 适合“查询多、修改少”的场景。

| 操作           | 时间复杂度   | 原因         |
| -------------- | ------------ | ------------ |
| get(index)     | O(1)         | 数组下标访问 |
| add(E)（尾部） | O(1)（均摊） | 可能触发扩容 |
| add(index, E)  | O(n)         | 需要移动元素 |
| remove(index)  | O(n)         | 需要移动元素 |
| contains()     | O(n)         | 线性遍历     |

Array是否线程安全？
: ArrayList 不是线程安全的。在多线程环境下同时进行 add 操作，可能导致数据覆盖、size 计算错误甚至扩容时数据丢失。如果需要线程安全，可以使用 Collections.synchronizedList() 包装，或者在并发读多写少场景下使用 CopyOnWriteArrayList。

#### LinkedList

底层数据结构
: LinkedList 底层是双向链表结构，每个节点包含三个部分：元素本身（item）、前驱指针（prev）和后继指针（next）。各节点在内存中不是连续存储，而是通过指针连接。内部维护了 first（头节点）和 last（尾节点），因此在头尾插入删除效率较高。同时在按索引查找时，会根据 index 判断从头还是从尾开始遍历，以减少遍历次数，但本质仍是顺序访问。
: 它更适用于插入删除频繁、不强调随机访问的场景，例如实现队列、双端队列（Deque）。如果业务中经常根据下标访问元素，则性能较差，因为每次都需要遍历链表。因此总体来说，LinkedList 适合“增删多、查询少”的场景。

| 操作                    | 时间复杂度 | 说明           |
| ----------------------- | ---------- | -------------- |
| get(index)              | O(n)       | 需要遍历链表   |
| add(E)（尾部）          | O(1)       | 直接修改尾指针 |
| add(index, E)           | O(n)       | 先定位，再插入 |
| remove(index)           | O(n)       | 先定位，再删除 |
| 插入/删除（已定位节点） | O(1)       | 仅修改前后指针 |
| contains()              | O(n)       | 线性遍历       |

### Set

常见的Set实现有HashSet和LinkedHashSet，HashSet和TreeSet。

#### HashSet

底层结构
: HashSet 底层其实就是一个 HashMap。源码中内部维护了一个 HashMap<E,Object>，存入的元素作为 key，而 value 是一个固定常量对象（PRESENT）。因此可以理解为：HashSet 本质是“只使用 key 的 HashMap”，它的所有操作（add、remove、contains）最终都是通过 HashMap 来完成的。

为什么元素不重复？
: 元素不重复是因为 HashMap 的 key 不允许重复。当调用 add() 时，本质是执行 map.put(e, PRESENT)。添加元素时会先计算 hashCode 定位桶位置，如果桶中已有元素，会先比较 hash，再调用 equals 比较内容，如果 equals 返回 true，则认为元素重复，不会加入集合。

时间复杂度
: 在理想情况下，HashSet 的 add、remove、contains 时间复杂度都是 O(1)，因为底层基于哈希表实现。但如果发生大量哈希冲突，时间复杂度可能退化为 O(n)。JDK8 之后，当链表长度达到 8 时会转换为红黑树结构，将最坏时间复杂度优化为 O(log n)。

#### LinkedHashSet

LinkedHashSet 的特点
: LinkedHashSet 底层是 LinkedHashMap（继承自 HashSet，本质还是哈希表），在 HashSet 的基础上增加了一条双向链表来维护元素的插入顺序。因此它既具备 HashSet 的 O(1) 查找效率，又能保证元素按照插入顺序遍历。它同样不允许重复元素，判断逻辑仍然依赖 hashCode 和 equals，只是多维护了一份顺序结构，内存开销略高于 HashSet。

和 HashSet 的用途区别
: HashSet 只保证元素唯一，不保证顺序，适合单纯去重、对遍历顺序没有要求的场景，性能略优、占用内存更少。而 LinkedHashSet 适用于既要去重又要保持插入顺序的场景，例如去重后还需要按原顺序输出数据。因此可以理解为：如果不关心顺序用 HashSet；如果需要稳定遍历顺序用 LinkedHashSet。

#### TreeSet

底层数据结构
: TreeSet 底层是 TreeMap，而 TreeMap 底层实现是红黑树（自平衡二叉搜索树）。在源码中可以看到 TreeSet 内部维护了一个 TreeMap，元素作为 key 存储。因此，TreeSet 本质是基于红黑树实现的有序集合。红黑树通过旋转和变色保证整棵树始终近似平衡，从而使查询、插入、删除的时间复杂度稳定在 O(log n)。

如何排序？
: TreeSet 的排序基于比较规则，而不是 hash。红黑树在插入元素时，会根据 compareTo 或 Comparator 的比较结果决定节点放在左子树还是右子树，因此元素始终保持有序。需要注意的是：TreeSet 判断元素是否重复也是依赖 compareTo/compare 结果是否为 0，而不是 equals。

TreeSet vs LinkedHashMap 区别
: TreeSet 主要用于需要自动排序或范围查询的场景，例如按大小排序存储数据、获取最小值/最大值、查找某个区间内的元素，因为它底层是红黑树，天然支持有序结构。而 LinkedHashMap 主要用于既要高效查找又要保持遍历顺序的场景，例如按照插入顺序输出数据或实现 LRU 缓存（访问顺序模式）。简单来说：TreeSet 强调“排序能力”，LinkedHashMap 强调“顺序稳定 + 哈希查询效率”。

## Map

Map 体系用于存储键值对（Key-Value），Key 不可重复，Value 可以重复。它本质上不是 Collection 的子接口，而是一个独立体系。常见实现包括：

- HashMap：基于哈希表（数组 + 链表 + 红黑树），查询效率高。
- LinkedHashMap：在 HashMap 基础上维护双向链表，保证插入顺序或访问顺序。
- TreeMap：基于红黑树实现，按 Key 排序。
- ConcurrentHashMap：线程安全的 HashMap，适用于并发环境。

### HashMap

底层结构
: HashMap 底层采用 数组 + 链表 + 红黑树 结构。核心是一个 Node[] 数组（桶数组），每个桶位置可能是：空、链表、或红黑树。当发生哈希冲突时，元素会以链表形式挂在同一个桶上；当链表长度 ≥ 8 且数组容量 ≥ 64 时，会转换为红黑树，以提升查询效率。因此 JDK8 后的结构是“数组 + 链表 + 红黑树”的组合结构。

JDK7 和 JDK8 区别
: JDK7 采用 数组 + 链表 结构，链表采用头插法，扩容时需要重新计算所有元素位置（rehash），在多线程环境下可能形成链表死循环。JDK8 引入红黑树优化，当链表长度达到 8 时转为红黑树；同时改为尾插法，并优化了扩容逻辑（高位判断法），不再需要完全重新计算 hash，从而解决了 JDK7 扩容时可能出现的死循环问题。

为什么要引入红黑树？
: 在极端情况下，如果大量元素哈希冲突落在同一个桶中，链表长度会很长，查询效率会退化为 O(n)。引入红黑树后，当链表长度 ≥ 8 时转换为红黑树，使最坏时间复杂度从 O(n) 优化为 O(log n)，从而保证查询性能更加稳定。

扩容机制
: HashMap 默认初始容量为 16，默认负载因子为 0.75。当元素个数超过 容量 × 负载因子 时触发扩容（即 16 × 0.75 = 12）。扩容为原来的 2 倍。JDK8 扩容时通过判断元素 hash 的高位是 0 还是 1，决定元素留在原索引还是移动到“原索引 + 旧容量”位置，从而避免重新计算 hash，提高效率。

为什么线程不安全？
: HashMap 不是线程安全的。在并发环境下，可能出现数据覆盖、丢失更新等问题。JDK7 中在扩容时甚至可能形成链表环形结构，导致死循环（CPU 100%）。JDK8 虽然优化了死循环问题，但仍然不是线程安全容器，因此并发环境下应使用 ConcurrentHashMap。

### LinkedHashMap

和 HashMap 的区别
: LinkedHashMap 本质上是 HashMap 的子类，底层仍然是 数组 + 链表 + 红黑树 结构，但在此基础上额外维护了一条双向链表来记录元素顺序。因此，HashMap 不保证遍历顺序，而 LinkedHashMap 可以保证元素按插入顺序或访问顺序进行遍历。代价是多维护指针，内存占用略高于 HashMap。

LRU 如何实现？
: LinkedHashMap 可以通过构造函数开启“访问顺序模式”（accessOrder = true），每次访问元素时都会将该节点移动到链表尾部。然后重写 removeEldestEntry() 方法，当元素数量超过设定容量时删除链表头节点（最久未使用的元素），即可实现 LRU（Least Recently Used）缓存机制。因此，LinkedHashMap = 哈希表查询效率 + 双向链表维护访问顺序，非常适合实现 LRU 缓存。

### TreeMap

底层结构
: TreeMap 底层是红黑树（自平衡二叉搜索树）。所有元素按照 Key 的比较规则进行排序存储。插入、删除、查找操作都会通过红黑树的旋转和变色机制保持树的平衡，因此时间复杂度稳定在 O(log n)。它不依赖 hash，而是依赖 Key 的比较结果决定元素在树中的位置。

和 HashMap 的区别
: HashMap 基于数组 + 链表 + 红黑树实现，依赖 hash 定位，查询平均时间复杂度 O(1)，但不保证顺序；而 TreeMap 基于红黑树实现，按 Key 自动排序，查询时间复杂度 O(log n)。简单来说：HashMap 强调查询效率，TreeMap 强调有序性。

### ConcurrentHashMap

JDK7 vs JDK8 区别
: JDK7 的 ConcurrentHashMap 采用 Segment 分段锁：整体是 Segment[] + HashEntry[]，每个 Segment 继承 ReentrantLock，写操作只锁住某个 Segment，从而实现并发；读操作大多无锁。JDK8 取消 Segment，结构更接近 HashMap：Node[] + 链表/红黑树，并发控制变为 CAS + synchronized（桶级别锁），锁粒度从“段”进一步细化到“桶/链表头节点”，并引入更高效的扩容迁移机制（transfer/ForwardingNode）。

为什么线程安全？
: 它的线程安全来自于“关键路径的原子性 + 可见性保证”：表（table）的初始化、桶位插入等关键步骤用 CAS 保证只有一个线程成功；发生哈希冲突需要在同一个桶内追加/替换节点时，用 synchronized 锁住该桶的头节点（bin lock） 来串行化修改，避免链表/树结构被并发破坏；读操作通常不加锁，通过 volatile/CAS 等保证看到的是一致的已发布数据，从而实现高并发下读多写少的性能。

CAS + synchronized 怎么结合？
: 典型 put 流程是“先 CAS，失败再加锁”：先通过 CAS 尝试把新节点放进空桶（无冲突路径，最快）；如果桶非空（发生冲突），再对桶头节点 synchronized (f) 加锁，在锁内完成链表/红黑树的查找、更新或插入；如果遇到扩容中的桶（ForwardingNode），当前线程会协助迁移（help transfer）。所以它不是“全程加锁”，而是 CAS 负责无冲突的快速写入 + synchronized 负责冲突桶内的结构安全，两者配合把锁粒度降到最低。

`map.put("java", map.get("java") + 1)` 为什么可能出错，但是 `merge` 不会？
: 因为它是“读 → 计算 → 写”三个独立步骤的复合操作，不具备原子性，多个线程可能同时读到相同的旧值，导致丢失更新；而 merge() 会把“读取旧值、执行计算、写回新值”这整个过程封装成一个原子操作，在内部通过 CAS + synchronized（桶级锁）控制并发，确保同一时刻只有一个线程能完成该 key 的更新，因此不会出现并发覆盖问题。

## Collections

Collection 是接口，Collections 是工具类，后者提供一系列静态方法操作集合。

- Collections.emptyList()：用于安全地返回一个不可修改的空集合，避免返回 null。
- Collections.singletonList(obj)：用于将单个元素快速包装成一个不可修改的 List。
- Collections.unmodifiableList(list)：用于给已有集合添加只读保护，防止外部代码修改集合内容。

| 分类     | 方法                       | 作用说明               |
| -------- | -------------------------- | ---------------------- |
| 排序     | sort(list)                 | 按自然顺序排序         |
| 排序     | sort(list, comparator)     | 按自定义规则排序       |
| 顺序操作 | reverse(list)              | 反转列表顺序           |
| 顺序操作 | shuffle(list)              | 随机打乱顺序           |
| 顺序操作 | swap(list, i, j)           | 交换两个位置元素       |
| 查找     | binarySearch(list, key)    | 二分查找（前提已排序） |
| 统计     | max(collection)            | 返回最大值             |
| 统计     | min(collection)            | 返回最小值             |
| 统计     | frequency(collection, obj) | 统计元素出现次数       |
| 特殊集合 | emptyList()                | 返回空集合（不可修改） |
| 特殊集合 | singletonList(obj)         | 返回单元素集合         |
| 特殊集合 | unmodifiableList(list)     | 返回不可修改视图       |

## 常用方法

### 初始化

可变容器的快速初始化：

| 容器                    | 快速初始化写法                                                 |
| ----------------------- | -------------------------------------------------------------- |
| ArrayList               | `List<Integer> list = new ArrayList<>();`                      |
| HashMap                 | `Map<Integer,Integer> map = new HashMap<>();`                  |
| HashSet                 | `Set<Integer> set = new HashSet<>();`                          |
| PriorityQueue（小根堆） | `PriorityQueue<Integer> pq = new PriorityQueue<>();`           |
| PriorityQueue（大根堆） | `PriorityQueue<Integer> pq = new PriorityQueue<>((a,b)->b-a);` |
| ArrayDeque              | `Deque<Integer> dq = new ArrayDeque<>();`                      |

of() 系列方法（如 List.of()、Set.of()、Map.of()）都是 Java 9 提供的静态工厂方法，用于快速创建真正的不可变集合；创建后的集合不能进行任何修改操作（如 add、remove、put 等都会抛出 UnsupportedOperationException），并且不允许存储 null 元素，通常适用于定义常量集合、配置数据或测试数据场景。

不可变容器的快速初始化：

| 方法              | 示例                              | 是否可修改  | 是否允许 null | 特点            |
| ----------------- | --------------------------------- | ----------- | ------------- | --------------- |
| `List.of()`       | `List.of(1,2,3)`                  | ❌ 不可修改 | ❌ 不允许     | 创建不可变 List |
| `Set.of()`        | `Set.of(1,2,3)`                   | ❌ 不可修改 | ❌ 不允许     | 不允许重复元素  |
| `Map.of()`        | `Map.of(1,"A",2,"B")`             | ❌ 不可修改 | ❌ 不允许     | 键不能重复      |
| `Map.ofEntries()` | `Map.ofEntries(Map.entry(1,"A"))` | ❌ 不可修改 | ❌ 不允许     | 适合多元素创建  |

### 容器方法

1. ArrayList

| 操作                   | 作用         | 时间复杂度 |
| ---------------------- | ------------ | ---------- |
| add(e)                 | 尾部添加     | O(1) 均摊  |
| add(i,e)               | 指定位置插入 | O(n)       |
| get(i)                 | 索引访问     | O(1)       |
| set(i,e)               | 修改         | O(1)       |
| remove(i)              | 删除         | O(n)       |
| size()                 | 长度         | O(1)       |
| Collections.sort(list) | 排序         | O(n log n) |

2. HashMap

| 操作                    | 作用     | 时间复杂度 |
| ----------------------- | -------- | ---------- |
| put(k,v)                | 存键值   | O(1)       |
| get(k)                  | 取值     | O(1)       |
| getOrDefault(k,d)       | 默认值   | O(1)       |
| containsKey(k)          | 判断存在 | O(1)       |
| remove(k)               | 删除     | O(1)       |
| merge(k,1,Integer::sum) | 计数统计 | O(1)       |
| entrySet()              | 遍历     | O(n)       |

3. HashSet

| 操作        | 作用     | 时间复杂度 |
| ----------- | -------- | ---------- |
| add(e)      | 添加     | O(1)       |
| remove(e)   | 删除     | O(1)       |
| contains(e) | 判断存在 | O(1)       |
| size()      | 元素数量 | O(1)       |

4. PriorityQueue

| 操作     | 作用     | 时间复杂度 |
| -------- | -------- | ---------- |
| offer(e) | 入堆     | O(log n)   |
| poll()   | 弹出堆顶 | O(log n)   |
| peek()   | 查看堆顶 | O(1)       |
| size()   | 堆大小   | O(1)       |

5. ArrayDeque

| 操作          | 作用     | 时间复杂度 |
| ------------- | -------- | ---------- |
| offerLast(e)  | 尾入队   | O(1)       |
| offerFirst(e) | 头入队   | O(1)       |
| pollFirst()   | 头出队   | O(1)       |
| pollLast()    | 尾出队   | O(1)       |
| peekFirst()   | 查看队头 | O(1)       |
| peekLast()    | 查看队尾 | O(1)       |
