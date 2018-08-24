---
layout:    post
title:      "Java容器"
subtitle:   "Java Collection"
date:       2018-06-22 12:00:00
author:     "scyhssm"
header-img: "img/Java.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Java
---

> Java容器主要包括Collection和Map两种，Collection又包含了List、Set以及Queue

## Collection
￼￼￼￼￼￼![JavaCollections](/img/java-collections.png)
1.Set
* HashSet：基于哈希实现，支持快速查找，但不支持有序性操作，例如根据一个范围查找元素的操作。并且失去了元素的插入顺序信息，也就是说使用 Iterator 遍历 HashSet 得到的结果是不确定的；
* TreeSet：基于红黑树实现，支持有序性操作，但是查找效率不如 HashSet，HashSet 查找时间复杂度为 O(1)，TreeSet 则为 O(logN)；
* LinkedHashSet：具有 HashSet 的查找效率，且内部使用链表维护元素的插入顺序。
2.List
* ArrayList：基于动态数组实现，支持随机访问；
* Vector：和 ArrayList 类似，但它是线程安全的；
* LinkedList：基于双向链表实现，只能顺序访问，但是可以快速地在链表中间插入和删除元素。不仅如此，LinkedList 还可以用作栈、队列和双向队列。
3.Queue
* LinkedList：可以用它来支持双向队列；
* PriorityQueue：基于堆结构实现，可以用它来实现优先队列。

## Map
![JavaMap](/img/Java-Map.tiff)
* HashMap：基于哈希实现；
* HashTable：和 HashMap 类似，但它是线程安全的，这意味着同一时刻多个线程可以同时写入 HashTable 并且不会导致数据不一致。它是遗留类，不应该去使用它。现在可以使用 ConcurrentHashMap 来支持线程安全，并且 ConcurrentHashMap 的效率会更高，因为 ConcurrentHashMap 引入了分段锁。
* LinkedHashMap：使用链表来维护元素的顺序，顺序为插入顺序或者最近最少使用（LRU）顺序。
* TreeMap：基于红黑树实现。

## 知识点
1.迭代器模式，Collection 实现了 Iterable 接口，其中的 iterator() 方法能够产生一个 Iterator 对象，通过这个对象就可以迭代遍历 Collection 中的元素。从 JDK 1.5 之后可以使用 foreach 方法来遍历实现了 Iterable 接口的聚合对象。

2.适配器模式，java.util.Arrays#asList() 可以把数组类型转换为 List 类型。如果要将数组类型转换为 List 类型，应该注意的是 asList() 的参数为泛型的变长参数，因此不能使用基本类型数组作为参数，只能使用相应的包装类型数组。

3.ArrayList，实现了 RandomAccess 接口，因此支持随机访问，这是理所当然的，因为 ArrayList 是基于数组实现的。添加元素时使用 ensureCapacityInternal() 方法来保证容量足够，如果不够时，需要使用 grow() 方法进行扩容，新容量的大小为 oldCapacity + (oldCapacity >> 1)，也就是旧容量的 1.5 倍。扩容操作需要调用 Arrays.copyOf() 把原数组整个复制到新数组中，这个操作代价很高，因此最好在创建 ArrayList 对象时就指定大概的容量大小，减少扩容操作的次数。删除元素需要调用 System.arraycopy() 将 index+1 后面的元素都复制到 index 位置上。Fail-Fast，modCount 用来记录 ArrayList 结构发生变化的次数。结构发生变化是指添加或者删除至少一个元素的所有操作，或者是调整内部数组的大小，仅仅只是设置元素的值不算结构发生变化。在进行序列化或者迭代等操作时，需要比较操作前后 modCount 是否改变，如果改变了需要抛出 ConcurrentModificationException。实际的使用当某一个线程通过iterator去遍历某集合的过程中，若该集合的内容被其他线程所改变了；那么线程访问集合时，就会抛出ConcurrentModificationException异常，产生fail-fast事件。fail-safe任何对集合结构的修改都会在一个复制的集合上进行修改，因此不会抛出ConcurrentModificationException。

4.Vector，同步，它的实现与 ArrayList 类似，使用了 synchronized 进行同步。

5.ArrayList 与 Vector
* Vector 是同步的，因此开销就比 ArrayList 要大，访问速度更慢。最好使用 ArrayList 而不是 Vector，因为同步操作完全可以由程序员自己来控制；
* Vector 每次扩容请求其大小的 2 倍空间，而 ArrayList 是 1.5 倍。

6.为了获得线程安全的 ArrayList，可以使用 Collections.synchronizedList(); 得到一个线程安全的 ArrayList。也可以使用 concurrent 并发包下的 CopyOnWriteArrayList 类。CopyOnWriteArrayList 是一种 CopyOnWrite 容器，读取元素是从原数组读取；添加元素是在复制的新数组上。读写分离，因而可以在并发条件下进行不加锁的读取，读取效率高，适用于读操作远大于写操作的场景。

7.LinkedList，基于双向链表实现，内部使用 Node 来存储链表节点信息。每个链表存储了 Head 和 Tail 指针。

8.ArrayList 与 LinkedList
* ArrayList 基于动态数组实现，LinkedList 基于双向链表实现；
* ArrayList 支持随机访问，LinkedList 不支持；
* LinkedList 在任意位置添加删除元素更快。

9.HashMap，内部包含了一个 Entry 类型的数组 table。Entry 就是存储数据的键值对，它包含了四个字段。从 next 字段我们可以看出 Entry 是一个链表，即数组中的每个位置被当成一个桶，一个桶存放一个链表，链表中存放哈希值相同的 Entry。也就是说，HashMap 使用拉链法来解决冲突。当桶的容量过大时会更改结构为红黑树。
￼
![Entry](/img/entry-struct.png)

为了让查找的成本降低，应该尽可能使得 N/M 尽可能小，因此需要保证 M 尽可能大，也就是说 table 要尽可能大，即桶的数量要增加。

HashMap 采用动态扩容来根据当前的 N 值来调整 M 值，使得空间效率和时间效率都能得到保证。

扩容使用 resize() 实现，需要注意的是，扩容操作同样需要把旧 table 的所有键值对重新插入新的 table 中，因此这一步是很费时的。

在进行扩容时，需要把键值对重新放到对应的桶上。HashMap 使用了一个特殊的机制，可以降低重新计算桶下标的操作。

对于一个 Key，它的哈希值如果在第 6 位上为 0，那么取模得到的结果和之前一样；如果为 1，那么得到的结果为原来的结果 + 8。HashMap 构造函数允许用户传入的容量不是 2 的 n 次方，因为它可以自动地将传入的容量转换为 2 的 n 次方。

10.从 JDK 1.8 开始，一个桶存储的链表长度大于 8 时会将链表转换为红黑树。红黑树中节点个数少于等6转换为链表。

11.HashMap 与 HashTable
* HashTable 使用 synchronized 来进行同步。
* HashMap 可以插入键为 null 的 Entry。
* HashMap 的迭代器是 fail-fast 迭代器。
* HashMap 不能保证随着时间的推移 Map 中的元素次序是不变的。

12.ConcurrentHashMap，存储结构
```
static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;
}
```
ConcurrentHashMap 和 HashMap 实现上类似，最主要的差别是 ConcurrentHashMap 采用了分段锁（Segment），每个分段锁维护着几个桶（HashEntry），多个线程可以同时访问不同分段锁上的桶，从而使其并发度更高（并发度就是 Segment 的个数）。Segment 继承自 ReentrantLock。
```
static final class Segment<K,V> extends ReentrantLock implements Serializable {

    private static final long serialVersionUID = 2249069246763182397L;

    static final int MAX_SCAN_RETRIES =
        Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;

    transient volatile HashEntry<K,V>[] table;

    transient int count;

    transient int modCount;

    transient int threshold;

    final float loadFactor;
}
```
final Segment<K,V>[] segments;

默认的并发级别为 16，也就是说默认创建 16 个 Segment。

static final int DEFAULT_CONCURRENCY_LEVEL = 16;
￼
![ConcurrentHashMap](/img/concurrent-hash-map.png)

每个 Segment 维护了一个 count 变量来统计该 Segment 中的键值对个数。transient int count;

在执行 size 操作时，需要遍历所有 Segment 然后把 count 累计起来。

ConcurrentHashMap 在执行 size 操作时先尝试不加锁，如果连续两次不加锁操作得到的结果一致，那么可以认为这个结果是正确的。

尝试次数使用 RETRIES_BEFORE_LOCK 定义，该值为 2，retries 初始值为 -1，因此尝试次数为 3。

如果尝试的次数超过 3 次，就需要对每个 Segment 加锁。

13.JDK1.8对ConcurrentHashMap改动，JDK 1.7 使用分段锁机制来实现并发更新操作，核心类为 Segment，它继承自重入锁 ReentrantLock，并发程度与 Segment 数量相等。JDK 1.8 使用了 CAS 操作来支持更高的并发度，在 CAS 操作失败时使用内置锁 synchronized。并且 JDK 1.8 的实现也在链表过长时会转换为红黑树。
