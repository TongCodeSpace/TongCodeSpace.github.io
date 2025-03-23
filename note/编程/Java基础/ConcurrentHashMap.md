---
tags:
  - 并发
  - Java基础
  - 集合
layout: post
author: tong
title: ConcurrentHashMap详解
---

# 一、ConcurrentHashMap 核心设计

1. 版本演变与核心思想
    - JDK 1.7：分段锁（Segment 数组 + HashEntry）。
    - JDK 1.8：CAS + synchronized 锁优化（数组 + 链表/红黑树）。
    - 核心目标：减少锁竞争，提高并发度。
2. 数据结构
    - Node 节点：链表结构（hash、key、value、next）。
    - TreeNode 与 TreeBin：红黑树结构（链表转树的阈值）。
    - ForwardingNode：扩容时的占位节点。
3. 核心成员变量
    - `table`：存储 Node 的数组，volatile 修饰保证可见性。
    - `sizeCtl`：控制表初始化和扩容的标记。
    - `transferIndex`：扩容时分配任务的索引。

# 二、核心方法实现原理（1.8 版本）

## 0. 核心成员变量

```Java
// 默认初始容量：16
private static final int DEFAULT_CAPACITY = 16;
// 默认负载因子：0.75（容量超过阈值触发扩容）
private static final float LOAD_FACTOR = 0.75f;
// 并发级别（兼容旧版本，JDK8已废弃但保留）
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
// 链表转红黑树的阈值：8
static final int TREEIFY_THRESHOLD = 8;
// 红黑树转链表的阈值：6
static final int UNTREEIFY_THRESHOLD = 6;
// 最小树化容量：64（数组长度需≥64才会树化）
static final int MIN_TREEIFY_CAPACITY = 64;
```

## 1 . Put 方法流程

***put () 方法***
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {  
    if (key == null || value == null) throw new NullPointerException();// 键值均不可为空  
    int hash = spread(key.hashCode());//计算 hash 值，扰动函数  
    int binCount = 0;  
    for (Node<K,V>[] tab = table;;) {  
        Node<K,V> f;  
        int n, i, fh;  
        if (tab == null || (n = tab.length) == 0)  
            tab = initTable();// 初始化表（懒加载）  
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { //i 元素在数组中的下标，f 所在位置的 node 元素  
            //f node 元素为空  
            if (casTabAt(tab, i, null,  
                    new Node<K,V>(hash, key, value, null)))  
                break;                   // 添加到下标为空的位置时不加锁，通过 CAS 进行插入，结束流程  
        }  
        else if ((fh = f.hash) == MOVED) //当前数组在扩容中  
            tab = helpTransfer(tab, f); // 帮助进行扩容  
        else {  
            V oldVal = null;  
            synchronized (f) { // 锁住节点  
                if (tabAt(tab, i) == f) { //再次判断防止已经被修改  
                    if (fh >= 0) { //正常 hash 值处理（）  
                        binCount = 1;  
                        for (Node<K,V> e = f;; ++binCount) {  
                            K ek;  
                            if (e.hash == hash &&  
                                    ((ek = e.key) == key ||  
                                            (ek != null && key.Equals (ek)))) {//找到对应的 key                                oldVal = e.val;  
                                If (! OnlyIfAbsent)  
                                    e.val = value;// 修改值  
                                Break;  
                            }  
                            Node<K,V> pred = e;  
                            if ((e = e.next) == null) {  
                                pred. Next = new Node<K,V>(hash, key,  
                                        Value, null); //添加到末尾  
                                Break;  
                            }  
                        }  
                    }  
                    Else if (f instanceof TreeBin) { // 已经树化，此时 fh = TREEBIN（-2）  
                        Node<K,V> p;  
                        BinCount = 2;  
                        if ((p = ((TreeBin<K,V>) f). PutTreeVal (hash, key,  
                                Value)) != null) { // 对树进行操作  
                            oldVal = p.val;  
                            If (! OnlyIfAbsent)  
                                p.val = value;  
                        }  
                    }  
                }  
            }  
            If (binCount != 0) { // 操作成功  
                If (binCount >= TREEIFY_THRESHOLD)  // 链表长度≥8，触发树化检查  
                    TreeifyBin (tab, i);  
                If (oldVal != null)  
                    Return oldVal;  
                Break;  
            }  
        }  
    }  
    AddCount (1 L, binCount); //// 更新元素计数  
    Return null;  
}
```

**核心步骤**
1. 计算哈希值定位桶位置。
2. 桶为空时通过 CAS 插入。
3. 桶非空时使用 synchronized 锁住链表头节点。
4. 链表转红黑树的阈值判断（链表长度 ≥8，数组长度 ≥64）。

>***TreeBin 的解析***
> `TreeBin` 是 `ConcurrentHashMap` 中用于管理红黑树的核心类，**本质是一个代理节点**，负责协调并发访问红黑树时的读写操作。以下是其设计思想的深度解析：
> **一、TreeBin 的定位与作用**
> 1. **替代 TreeNode 的根节点**
> 	- 当链表长度超过阈值（默认 8）且数组长度 ≥64 时，链表会转换为红黑树。
> 	- 转换后，原链表的头节点会被替换为一个 `TreeBin` 对象，而真正的红黑树根节点由 `TreeBin` 内部维护。
> 2. **职责分离**
> 	- **TreeNode**：仅负责红黑树节点的数据存储和树结构操作（如插入、删除）。
> 	- **TreeBin**：负责并发控制（如读写锁）、状态维护（如锁标志）和树的遍历。
> **二、核心设计思想**
> 1. **读写锁分离**
>	- **读操作无锁化**：通过 `volatile` 状态标志和 `CAS` 实现乐观读（类似 `StampedLock` 的设计）。
>	- **写操作加锁**：通过 `synchronized` 锁住 `TreeBin` 对象，确保写操作的原子性。

***表的初始化***
```java
private final Node<K,V>[] initTable () {
    Node<K,V>[] tab; int sc;
    While ((tab = table) == null || tab. Length == 0) { // 循环直到表初始化完成
        If ((sc = sizeCtl) < 0) // 检查 sizeCtl 是否表示其他线程正在初始化
            Thread.Yield ();     // 让出 CPU，自旋等待其他线程完成初始化
        else if (U.compareAndSwapInt (this, SIZECTL, sc, -1)) { // CAS 抢锁
            Try {
                If ((tab = table) == null || tab. Length == 0) { // 双重检查
                    Int n = (sc > 0) ? Sc : DEFAULT_CAPACITY;    // 确定初始容量
                    @SuppressWarnings ("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[]) new Node<?,?>[n]; // 创建新数组
                    Table = tab = nt;   // 赋值给 table
                    Sc = n - (n >>> 2); // 计算扩容阈值（n * 0.75）
                }
            } finally {
                SizeCtl = sc;          // 更新 sizeCtl 为扩容阈值
            }
            Break;                     // 初始化完成，退出循环
        }
    }
    Return tab; // 返回初始化后的数组
}

```

|**字段/变量**|**含义**|
|---|---|
|`table`|ConcurrentHashMap 的核心数组，存储链表的头节点或红黑树的根节点（延迟初始化）。|
|`sizeCtl`|控制标识符，用于控制表的初始化和扩容。<br>- **负数**：表示正在初始化或扩容（-1 表示初始化中）。<br>- **正数**：表示下一次扩容的阈值。|
|`DEFAULT_CAPACITY`|默认初始容量（值为 16）。|
|`U`|`Unsafe` 类的实例，用于执行 CAS（Compare-And-Swap）原子操作。|
|`SIZECTL`|`sizeCtl` 字段的内存偏移量（通过 `Unsafe` 操作该字段）。|
|`tab`|临时变量，指向当前表的引用。|
|`sc`|临时变量，保存 `sizeCtl` 的当前值。|
|`n`|临时变量，表示表的初始容量。|
|`nt`|临时变量，表示新创建的数组。|

## 2 . Get 方法流程

```Java
Public V get (Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    Int h = spread (key.HashCode ()); // 计算哈希
    If ((tab = table) != null && (n = tab. Length) > 0 &&
        (e = tabAt (tab, (n - 1) & h)) != null) { // 定位桶
        if ((eh = e.hash) == h) { // 头节点匹配
            if ((ek = e.key) == key || (ek != null && key.Equals (ek)))
                return e.val;
        } else if (eh < 0) // 红黑树或 ForwardingNode
            return (p = e.find (h, key)) != null ? p.val : null;
        while ((e = e.next) != null) { // 遍历链表
            if (e.hash == h && ((ek = e.key) == key || (ek != null && key.Equals (ek))))
                return e.val;
        }
    }
    Return null;
}

```
1. 无锁设计：依赖 volatile 的可见性读取数据。
2. 处理扩容时的 ForwardingNode 转发逻辑。

**ForwardingNode find () 逻辑**
```java
Node<K,V> find (int h, Object k) {
    // 从新数组（nextTable）中重新定位并查找
    outer: for (Node<K,V>[] tab = nextTable;;) {
        Node<K,V> e; int n;
        If (k == null || tab == null || (n = tab. Length) == 0 ||
            (e = tabAt (tab, (n - 1) & h)) == null)
            Return null;
        For (;;) {
            Int eh; K ek;
            if ((eh = e.hash) == h && ((ek = e.key) == k || (ek != null && k.equals (ek))))
                Return e; // 直接命中节点
            If (eh < 0) { // 遇到特殊节点（如新的 ForwardingNode）
                If (e instanceof ForwardingNode) {
                    tab = ((ForwardingNode<K,V>) e). NextTable; // 继续跳转到下一层新数组
                    Continue outer; // 跳出内层循环，继续外层循环
                }
                Else
                    return e.find (h, k); // 其他特殊节点（如 TreeBin）的查找逻辑
            }
            if ((e = e.next) == null) // 遍历链表
                Return null;
        }
    }
}

```

## 3 . Size 方法实现
```java
Public int size () {
    Long n = sumCount ();
    return ((n < 0L) ? 0 : (n > (long) Integer. MAX_VALUE) ? Integer. MAX_VALUE : (int) n);
}

Final long sumCount () {
    CounterCell[] as = counterCells; CounterCell a;
    Long sum = baseCount;
    If (as != null) {
        For (int i = 0; i < as. Length; ++i) {
            If ((a = as[i]) != null)
                sum += a.value; // 累加所有分块计数器
        }
    }
    Return sum;
}

```
1. 多线程并发更新时的统计策略（baseCount + CounterCell 分段计数）。
2. 最终一致性设计，非强一致性。

## 4 . 扩容机制（transfer）

```java
private final void transfer (Node<K,V>[] tab, Node<K,V>[] nextTab) {
    Int n = tab. Length, stride;
    // 计算每个线程处理的区间大小（默认最小步长为 16）
    If ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        Stride = MIN_TRANSFER_STRIDE;
    // 初始化新数组（若未初始化）
    If (nextTab == null) {
        Try {
            Node<K,V>[] nt = (Node<K,V>[]) new Node<?,?>[n << 1];
            NextTab = nt;
        } catch (Throwable ex) {
            SizeCtl = Integer. MAX_VALUE;
            Return;
        }
        NextTable = nextTab;
        TransferIndex = n; // 初始迁移索引为旧数组长度
    }
    Int nextn = nextTab. Length;
    // 创建占位节点（ForwardingNode）
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    Boolean advance = true;
    Boolean finishing = false; // 标记是否完成迁移
    // 主循环：处理每个区间的迁移任务
    For (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        // 步骤 1：确定当前线程处理的区间 [bound, i]
        While (advance) {
            Int nextIndex, nextBound;
            If (--i >= bound || finishing)
                Advance = false;
            Else if ((nextIndex = transferIndex) <= 0) {
                I = -1;
                Advance = false;
            }
            // CAS 更新迁移索引（transferIndex）
            else if (U.compareAndSwapInt (this, TRANSFERINDEX, nextIndex,
                      NextBound = (nextIndex > stride ? NextIndex - stride : 0))) {
                Bound = nextBound;
                I = nextIndex - 1;
                Advance = false;
            }
        }
        // 步骤 2：检查迁移是否完成
        if (i < 0 || i >= n || i + n >= nextn) {
            Int sc;
            If (finishing) { // 完成迁移，更新全局状态
                NextTable = null;
                Table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1); // 新阈值为新容量的 0.75 倍
                Return;
            }
            // 当前线程完成任务，更新 sizeCtl 中的线程计数
            if (U.compareAndSwapInt (this, SIZECTL, sc = sizeCtl, sc - 1)) {
                If ((sc - 2) != resizeStamp (n) << RESIZE_STAMP_SHIFT)
                    Return;
                Finishing = advance = true;
                I = n; // 最后检查所有桶
            }
        }
        // 步骤 3：处理空桶（标记为已迁移）
        Else if ((f = tabAt (tab, i)) == null)
            Advance = casTabAt (tab, i, null, fwd);
        // 步骤 4：处理已迁移的桶（跳过）
        else if ((fh = f.hash) == MOVED)
            Advance = true;
        // 步骤 5：迁移非空桶（链表或红黑树）
        Else {
            Synchronized (f) { // 锁住当前桶的头节点
                If (tabAt (tab, i) == f) { // 双重检查防止其他线程修改
                    Node<K,V> ln, hn;
                    If (fh >= 0) { // 普通链表节点
                        // 高低位拆分逻辑
                        Int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            If (b != runBit) {
                                RunBit = b;
                                LastRun = p;
                            }
                        }
                        If (runBit == 0) {
                            Ln = lastRun;
                            Hn = null;
                        } else {
                            Hn = lastRun;
                            Ln = null;
                        }
                        // 构建高低位链表
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            If ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            Else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // 将链表存入新数组
                        SetTabAt (nextTab, i, ln);
                        SetTabAt (nextTab, i + n, hn);
                        SetTabAt (tab, i, fwd); // 标记旧桶为已迁移
                        Advance = true;
                    } else if (f instanceof TreeBin) { // 红黑树迁移（类似链表逻辑）
                        // 省略树节点迁移代码...
                    }
                }
            }
        }
    }
}

```
 **1. 初始化新数组与分配任务**
- **新数组创建**：若 `nextTab` 为空，创建大小为旧数组 2 倍的新数组。
- **迁移索引分配**：通过 `transferIndex` 和 `stride` 将迁移任务分段，每个线程处理一段区间（如旧数组长度为 64，`stride=16`，则第一个线程处理索引 48~63 的桶）。

**2. 处理空桶与已迁移桶**
- **空桶标记**：若当前桶为空，直接标记为 `ForwardingNode`。
- **跳过已迁移桶**：若当前桶已被迁移（`hash=MOVED`），跳过处理。

**3. 迁移非空桶（链表与红黑树）**

- **锁住头节点**：通过 `synchronized` 锁住当前桶的头节点，防止并发修改。
- **高低位拆分**：
    - 根据哈希值的某一位（`hash & n`）将链表拆分为高位链和低位链。
    - **低位链**：存放在新数组的 `i` 位置。
    - **高位链**：存放在新数组的 `i + n` 位置。
- **树节点迁移**：逻辑类似链表，但需处理红黑树结构（代码略）。

**4. 完成迁移与状态更新**
- **全局状态检查**：当所有线程完成迁移后，更新 `table` 为新数组，并计算新的扩容阈值（`sizeCtl = 0.75 * newCapacity`）。

# 三、并发控制策略

- **多线程应用场景**：初始化竞争、并发写入、协同扩容、分片计数、无锁读取。
- **性能提升关键**：
    - **锁粒度细化**（桶级别锁 vs 全局锁）。
    - **无锁化设计**（CAS + volatile）。
    - **任务分治**（分片、分块）。
- **适用场景**：高并发读写、大规模数据存储（如缓存、实时计算）。
**设计哲学**：通过精细化并发控制，最大化多线程协作效率，在保证线程安全的前提下，最小化性能损耗。

# 四、与其他并发容器的对比

1. ConcurrentHashMap vs Hashtable
    - 锁粒度对比（分段锁 vs 全局锁）。
    - 性能差异（高并发下吞吐量）。
2. ConcurrentHashMap vs Collections. SynchronizedMap
    - 实现方式（CAS + synchronized vs 包装器模式）。
    - 适用场景（读多写少 vs 简单并发场景）。
3. ConcurrentHashMap vs CopyOnWriteArrayList
    - 适用场景（高频写 vs 读多写极少）。

# 五、JDK 版本差异与优化

| **特性**    | **JDK 7 分段锁 (Segment)** | **JDK 8 TreeBin**  |
| --------- | ----------------------- | ------------------ |
| **锁粒度**   | 段级别（多个桶共享一个锁）           | 树级别（每个红黑树单独锁）      |
| **读并发能力** | 读操作仍需获取段锁               | 读操作无锁（乐观读）         |
| **写并发能力** | 段内串行化                   | 树级别细粒度锁            |
| **数据结构**  | 链表                      | 红黑树 + 链表退化机制       |
| **原理**    | 分段锁                     | CAS + Synchronized |

1. JDK 1.7 到 JDK 1.8 的改进
    - 分段锁到 synchronized + CAS 的演进。
    - 红黑树优化链表查询效率。
    - 扩容性能提升（多线程协同）。体现在 `helpTransfer ()` 方法上
2. JDK 11+ 的优化
    - 计数器优化（LongAdder 思想的应用）。
    - 内部代码结构的简化与性能调优。

# 六、常见问题与高频面试题

***1. ConcurrentHashMap 如何保证线程安全？***

**核心机制**：

- **CAS（无锁化操作）**：用于空桶插入、计数器更新（如 `sizeCtl`）等场景，避免锁竞争。
- **synchronized（细粒度锁）**：锁住链表或红黑树的头节点，保证同一桶内的写操作原子性。
- **volatile 变量**：如 `table` 数组和节点的 `val`、`next` 字段，保证内存可见性。
- **多线程协作扩容**：通过 `ForwardingNode` 和分段迁移任务，允许读写操作与扩容并行。

**现实场景**：

- 多个线程同时插入不同桶时，无需锁竞争（如 10 个线程操作 10 个不同的桶）。
- 多个线程插入同一桶时，通过锁住头节点保证安全（如秒杀系统中同一商品的库存更新）。

**常见误区**：

- 误认为 `ConcurrentHashMap` 的读操作完全无锁（遇到 `ForwardingNode` 时可能需要锁）。
**结论**：通过 **CAS + synchronized + volatile + 协作式扩容** 实现高并发下的线程安全。

***2. JDK 1.8 中为什么用 synchronized 替代分段锁？***

**核心原因**：

- **锁粒度更细**：从段级别（多个桶共享一个锁）细化到桶级别（每个桶单独锁），减少竞争。
- **synchronized 优化**：JDK 1.6 后引入偏向锁、轻量级锁，性能接近 `ReentrantLock`，且代码更简洁。
- **红黑树支持**：树化后的平衡操作需要更细粒度的锁控制。

**对比场景**：

- JDK 1.7：16 个段，最多支持 16 个线程并发写入。
- JDK 1.8：数组长度为 64 时，最多支持 64 个线程并发写入（每个桶独立锁）。

**常见误区**：

- 认为 `synchronized` 性能一定比 `ReentrantLock` 差（实际在低竞争下更优）。

**结论**：synchronized **锁粒度更细、性能更优、代码更简单**，适应高并发需求。

***3. Size () 方法为什么不是绝对准确的？***

**核心机制**：

- **分片计数**：通过 `baseCount` + `CounterCell[]` 分散计数更新竞争。
- **弱一致性**：遍历 `CounterCell` 时，其他线程可能正在更新，导致统计值是某一时刻的近似值。

**现实场景**：

- 高并发插入时，`size ()` 返回的值可能与实际相差几百甚至几千（但误差可控）。

**常见误区**：

- 误用 `size ()` 做精确的业务逻辑判断（如库存校验），应改用原子操作或外部锁。

**结论**：为 **性能** 牺牲强一致性，`size ()` 适用于监控场景，不适用于精确控制。

***4. ConcurrentHashMap 在扩容时如何保证读写正常？***

**核心机制**：

- **ForwardingNode 转发**：已迁移的桶标记为 `ForwardingNode`，读操作自动跳转到新数组。
- **多线程协作迁移**：多个线程分段迁移数据（如线程 A 处理桶 1~~16，线程 B 处理 17~~32）。
- **读写并行**：写操作遇到未迁移的桶时，先协助迁移再插入。

**现实场景**：

- 扩容期间，用户仍然可以正常查询数据（如在线服务升级时不停机）。

**常见误区**：

- 认为扩容期间读写会被完全阻塞（实际只有迁移当前桶时短暂加锁）。

**结论**：通过 **转发节点 + 协作迁移** 实现扩容期间的读写高可用。

***5. 链表转红黑树的阈值为什么是 8？***

**核心原因**：

- **泊松分布**：哈希冲突达到 8 的概率极低（约千万分之一），树化代价可控。
- **平衡性能**：链表查询复杂度为 `O (n)`，红黑树为 `O (log n)`，冲突严重时性能提升显著。
- **退化阈值（6）**：避免频繁树化和链表化（如阈值设为 7 会导致频繁转换）。

**现实场景**：

- 在哈希函数设计合理时，几乎不会触发树化（如用户 ID 哈希均匀分布）。

**常见误区**：

- 认为链表越长越好（实际链表过长会严重影响查询性能）。

**结论**：基于 **统计学概率与性能平衡**，阈值设为 8 是工程经验的最优解。


---

[back](../编程相关文章汇总)

[home](../../../index)