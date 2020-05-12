集合框架是Java的重要组成部分，其中HashMap的实现原理及源码是经常被考察到的一个点。这里就对HashMap的实现原理做一个解析。

## 数据结构回顾
我们常用的基础数据结构大概有下面4种: 数组，线性链表，二叉树，哈希表。因为各自的实现结构不同，其对于新增，查找等不同操作的性能也不同。这里做一个简单的回顾。

**数组**: 采用一段连续的存储单元来存储数据。对于指定下标的查找，复杂度为O(1); 通过给定值查找，需要遍历数组，时间复杂度为O(n); 当然对于有序数组，可以采取二分查找等方式将复杂度降为O(logn); 对于插入或删除操作，因为涉及到数组元素的移动，其平均复杂度也为O(n)。

**线性链表**: 对于链表的新增，删除等操作（在找到指定操作位置后），仅需处理结点间的引用即可，时间复杂度为O(1)，而查找操作需要遍历链表逐一进行比对，复杂度为O(n)。

**二叉树**：对一棵相对平衡的有序二叉树，对其进行插入，查找，删除等操作，平均复杂度均为O(logn)。

**哈希表**: 相比上述几种数据结构，在哈希表中进行添加，删除，查找等操作，性能十分之高，不考虑哈希冲突的情况下，仅需一次定位即可完成，时间复杂度为O(1)。

哈希表的主干是数组，只不过我们在添加或查找的时候，使用了一个**哈希函数**来计算数据需要存储的位置。是先通过哈希函数计算出实际的存储地址，然后从数组中对应的地址取出来即可。

#### 哈希冲突
当你使用哈希函数计算地址的时候，会出现两个不同的元素，最终存储的地址一样的情况。这就是所谓的哈希冲突。哈希函数的设计很重要，好的哈希函数能够尽可能的保证计算简单和散列地址分布均匀。但是，再好的哈希冲突也不能完全保证得到的地址不发生冲突，为了解决冲突，我们用链表的形式来存储相同地址的不同元素。也就是**数组+链表**的方式。HashMap就是采用这种方式来解决冲突。

## HashMap实现原理
(这里以最新的JDK1.8的源码来分析)
随便打开IDE，找到HashMap类的源码文件，打开后首先映入眼帘的是一段开发者写的Implementation Notes. 这里截取前几句。大意是告诉你一般情况下，它会是一个哈希表的实现。可是如果这个map变得很大的话，会转换成TreeNode的实现。
>
* Implementation notes.
* This map usually acts as a binned (bucketed) hash table, but
* when bins get too large, they are transformed into bins of
* TreeNodes, each structured similarly to those in
* java.util.TreeMap.

这里我们还是以主要讨论一般情况，也就是哈希表的实现。

HashMap的主干是一个名叫table的Node数组, 每个Node包含一个key-value对。
```
transient Node<K,V>[] table;
```

Node是HashMap中的一个静态内部类
```
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

    // .. 省略了构造方法, getter/setter, toString()等方法
}
```
很容易看出，HashMap是由数组+链表组成的。数组是主体，链表则是为了解决哈希冲突而存在的。Node结构中key和value自然就是写入数据的键值，next是链表中指向下一个数据的指针，hash存放的是当前key的hashcode。正常存储或查找的步骤是，先通过哈希函数确定对应的存储地址，也就是数组下标，然后遍历链表来确定添加的位置或查找的结果。

接下来分析HashMap如何添加/查找，也就是如何实现put/get方法。

### put方法
这里直接贴上源码，为了方便指出位置，我直接在源码上注释来解释分析put方法。

```
public V put(K key, V value) {
    /* put函数的具体实现在putVal()函数里，这里值得注意的是第一个参数传进去了hash(key)的运算结果。有关hash()函数具体实现会在后面讲到。 */
    return putVal(hash(key), key, value, false, true);
}

/* putVal() 是put函数具体的实现方法。*/
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;

    /* 这里先判断table数组是否为空，为空的话就调用resize()方法初始化。resize()方法有两个功能，一是初始化数组，二是扩大数组容量为之前的两倍。这里调用显然是为了初始化数组。 */
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;

    /* 这里用 (n-1)&hash 计算出存储的数组下标，判断这个位置的Node是否为空。如果为空就newNode()一个出来, 那么put的数据最终就存在了这里 */
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        /* 如果不为空的话，先判断第一个Node的key是否与即将添加的数据的key相等，如果相等的话则拿出来存入变量e */
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            /* 这里是为了处理TreeNode结构下的逻辑，我们这次讨论先忽略 */
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            /* 这下面就是链表的查找逻辑了，遍历链表看看有没有key相等的情况存在。*/
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    /* 链表遍历完毕也没有相等的key, 那么就直接newNode(), 将新数据放在链表的尾部 */
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        /* 这里是1.8实现的一个优化，如果链表太长，则会将链表转换成红黑树的结构。 */
                        treeifyBin(tab, hash);
                    break;
                }

                /* 这里是在链表中找到了相同的key的数据e, 那么就break。 */
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }

        /* 这里是处理已经存在的数据e, 更新旧数据的e.value为新的value */
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }

    /** 若是成功添加了一个Node, 则modCount会加1，(modCount记录HashMap被修改的次数)。并且需要根据threhold来判断是否需要resize()扩容。这几个变量在后面也会讲到。 */
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
put函数用于向HashMap添加数据。如果key不存在就添加，如果key存在就更新对应的value值。以上代码应该很清晰的展现了如何实现这个功能。值得留意的是中间扩容的细节，以及链表太长转换为红黑树的思路。


### get方法
接下来分析get方法，同样的我会贴上源码，并且通过注释的方式分析。
```
/* get方法比较容易，先调用getNode()去找Node数据是否存在，如果不存在就返回null,存在就返回value. */
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}


final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;

    /* 如果table数组为空， 或者存储地址(通过n-1&hash计算得出)上的Node为空，则直接返回null. */
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {

        /* 如果该位置第一个Node就是要找的key，则返回这个node. */
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;

        /* 链表的遍历查找，找出是否存在所需的key的Node。 */
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                // 处理TreeNode实现的HashMap下的逻辑，我们这次讨论先忽略。
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```
get方法的功能是根据key去查找数据。如果key存在则返回value，不存在则返回null。以上便是get方法的源码的实现。

在阅读上面两个方法的源码的时候，你会发现除了put/get的功能实现外，还有其它的辅助方法(比如hash方法), 其它的成员变量以及字段(比如threshold), 性能优化的逻辑(比如resize扩容，链表转红黑树等)。接下来讲我将对着源码，看看它们具体的细节。

### hash()方法
```
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
这也就是hash方法，为了让hash结果更均匀的分布，同时也不消耗太多的算力。

### 重要字段
```
/**
* 默认初始容量16，必须是2的次幂
*/
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

/**
* 最大容量, 必须是2的次幂，并小于1<<30
*/
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
* 默认负载因子0.75, 这里举个例子解释下负载因子。一般来说，HashMap给定的默认容量为 16，负载因子为 0.75。Map 在使用过程中不断的往里面存放数据，当数量达到了 16 * 0.75 = 12 就需要将当前 16 的容量进行扩容，而扩容这个过程涉及到 rehash、复制数据等操作，所以非常消耗性能。因此负载因子决定了何时需要扩容操作。
*/
static final float DEFAULT_LOAD_FACTOR = 0.75f;

/**
* 判断是否需要将链表转换为红黑树的阈值
*/
static final int TREEIFY_THRESHOLD = 8;

```

## 为什么HashMap数组长度一定要是2的次幂？
我们能在代码注释里看到，HashMap是要求容量是2的次幂。有两个原因：

1. HashMap中计算数组下标的方法是:
```
index = (length - 1) & hash
```
其中hash是hash(key)计算出的hash值，length是数组的长度。通常我们取模的计算方式是hash % length，来得到数组下标。但是，当length长度是2的次幂的时候，上面位运算的结果会与取模的结果相等。因此，使用位运算可以避免取模的预算，从而提高计算速度。

2. 在resize()的方法上，有这么一段注释。
```
/**
* Initializes or doubles table size.  If null, allocates in
* accord with initial capacity target held in field threshold.
* Otherwise, because we are using power-of-two expansion, the
* elements from each bin must either stay at same index, or move with a power of two offset in the new table.
*
* @return the table
*/

final Node<K,V>[] resize() {
    ...
}
```
意思是，resize()方法能够初始化和加倍扩容数组。而由于数组的长度只能是2的次幂，在扩容后元素新的位置能够保证在与之前相同的位置，或者下标移动2的次幂的情况。这样能够减少扩容rehash以及数组移动元素带来的影响。


## 线程不安全
最后，线程安全性也是HashMap经常被问及的点。看完源码后这个答案是显而易见的，线程不安全。源码中没有任何锁及其它同步机制，多个线程同时修改一个HashMap，若是线程调度发生在某个条件判断之后，真正修改之前，那么就很容易产生线程安全问题。

CocurrentHashMap能够保证线程安全，之后也会分析它的源码，来看看它是如何保证线程安全的。

## 小结
这篇文章阅读了HashMap的源码。平时可能用HashMap比较多，但源码很少看。HashMap也是经常会被考察到的点。这次就当是一次学习，无论是对面试，还是提高自己都有好处。

