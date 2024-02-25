---
title: HashMap
date: 2024-02-25 18:35:20
tags:
    - Java
    - Java 基础
description: HashMap,Java 基础
keywords: HashMap,Java,Java 基础
---

<div style="background: #f0f0f0;font-style: italic;color: #999;font-weight: bold;">更多 Java 基础文章可见 {% post_link java_foundation/java_foundation %} </div>

<!-- more -->


## 一，HashMap 的实现

因为HashMap是基于key的hashCode值来存储value的，所以遍历HashMap不会保证它的顺序和插入时的顺序一致。

> 注意：HashMap允许key为null，但是只允许有一个key为null。

java7和java8在实现HashMap上有所区别，当然java8的效率要更好一些，主要是java8的HashMap在java7的基础上增加了红黑树这种数据结构，使得在桶里面查找数据的复杂度从O(n)降到O(logn)，当然还有一些其他的优化，比如resize的优化等。

HashMap 是线程不安全的，在并发环境下应该使用 ConcurrentHashMap。

### 1.1 HashMap 的数据结构

HashMap在实现上使用了数组+链表+红黑树三种数据结构。

<span style="color: green">由于寻找一个完美且通用的’哈希’函数以保证所有key的’散列值’都不一样几乎是不可能的。因此，为了解决哈希冲突，HashMap使用’拉链法’来解决碰撞。将所有’散列值’一样的 node 通过一个链表来维护。这样就使得 node 查询的时间复杂度变为了 O(N)（N为链表的长度）。而 table 数组的长度越小，产生冲突的可能性就越大，这样时间复杂度就越大；但如果为了减小这个时间复杂度，一味的增大 table 数组的大小，就会导致空间复杂度的大大增加。因此为了平衡’时间复杂度’和’空间复杂度’两个相矛盾的指标。HashMap 通过 factor 和’红黑树’来对其进行了优化。
首先，HashMap保证了一个合理的数组大小，以至于不会浪费太多的空间来满足时间的需求。同时，当链表的长度>8时，将链表转换为’红黑树’来实现时间复杂度从 O(N) -> O(logN) 的转变。</span>

HashMap的实现使用了一个数组，每个数组项里面有一个链表的方式来实现，因为HashMap使用key的hashCode来寻找存储位置，不同的key可能具有相同的hashCode，这时候就出现哈希冲突了，也叫做哈希碰撞，为了解决哈希冲突，有开放地址方法，以及链地址方法。HashMap的实现上选取了链地址方法，也就是将哈希值一样的entry保存在同一个数组项里面，可以把一个数组项当做一个桶，桶里面装的entry的key的hashCode是一样的。我们知道，链表上的查询复杂的为O(N)，当这个N很大的时候也就成了瓶颈，所以HashMap在链表的长度大于8的时候就会将链表转换为红黑树这种数据结构，红黑树的查询效率高达O(lgN)，也就是说，复杂度降了一个数量级，完全可以适用于实际生产环境。

<img src="https://p.ipic.vip/fd3fki.png" style="display: block;margin-left: auto;margin-right: auto;width: 70%;"/>  

<br>

其中有一个非常重要的数据结构Node<K,V>，这就是实际保存我们的key-value对的数据结构。一个Node就是一个链表节点，也就是我们插入的一条记录。

~~~java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;    // hash字段用来定位桶的索引位置
    final K key;
    V value;
    Node<K,V> next;    // 指向链表的下一个节点
}
~~~

<br>

### 1.2 HashMap 实现思路

① 定位 node 在 HashMap 的 table 数组中的索引。  
通过对 key 进行’哈希’，得到一个’散列值’，并将该’散列值’与 table 数组长度取模，得到 node 在 table 数组的索引。  
② 根据定位到的索引，进行’插入’、’删除’、’查询’、’修改’操作  
③ 在‘插入’操作完成后，查看 HashMap 中元素是否超过了阈值，若超过了，则进行扩容操作。  
threshold = capacity * factor；capacity 默认为 16；factor 默认为 0.75。
当进行扩容时，默认会扩大为原容量的2倍。

<br>

## 二，HashMap 源码解析

### 2.1 重要属性

#### HashMap中元素个数
~~~java
/**
 * The number of key-value mappings contained in this map.
 */
transient int size;
~~~
<br>

#### 默认容量
HashMap 的默认容量大小为 16。
HashMap 的容量必须为 2 的 n 次方。
~~~java
/**
 * The default initial capacity - MUST be a power of two.
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
~~~
<br>

#### 最大容量
~~~java
/**
 * The maximum capacity, used if a higher value is implicitly specified
 * by either of the constructors with arguments.
 * MUST be a power of two <= 1<<30.
 */
static final int MAXIMUM_CAPACITY = 1 << 30;
~~~
<br>

#### ‘链表’转换为’红黑树’的阈值
当’链表’的长度 >= 8，将’链表’转换为’红黑树’
~~~java
/**
 * The bin count threshold for using a tree rather than list for a
 * bin.  Bins are converted to trees when adding an element to a
 * bin with at least this many nodes. The value must be greater
 * than 2 and should be at least 8 to mesh with assumptions in
 * tree removal about conversion back to plain bins upon
 * shrinkage.
 */
static final int TREEIFY_THRESHOLD = 8;
~~~
<br>

#### 负载因子
用于计算扩容阈值用的。默认为 0.75
~~~java
/**
 * The load factor used when none specified in constructor.
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;
~~~
<br>

#### ‘红黑树’退化为’链表’的阈值
与’TREEIFY_THRESHOLD’相对应，如果‘红黑树’里的元素数目小于’UNTREEIFY_THRESHOLD’，‘红黑树’就退化成一个’链表’
使用场景：在 resize 后，每个桶中冲突的元素变少了，因此是可能从扩容前的’红黑树’退化为扩容后的’链表’来存储的。
~~~java
/**
 * The bin count threshold for untreeifying a (split) bin during a
 * resize operation. Should be less than TREEIFY_THRESHOLD, and at
 * most 6 to mesh with shrinkage detection under removal.
 */
static final int UNTREEIFY_THRESHOLD = 6;
~~~
<br>

#### 初始化’红黑树’的最小容量阈值
当’链表’的长度达到’TREEIFY_THRESHOLD’时，但是，table 数据的长度小于’MIN_TREEIFY_CAPACITY’，则此时，不将’链表’升级为’红黑树’，而是对整个HashMap进行扩容操作。
~~~java
/**
 * The smallest table capacity for which bins may be treeified.
 * (Otherwise the table is resized if too many nodes in a bin.)
 * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
 * between resizing and treeification thresholds.
 */
static final int MIN_TREEIFY_CAPACITY = 64;
~~~

<br>

### 2.2 重要方法

#### 确定 HashMap 的容量大小
因为 HashMap 的容量必须为 2 的 n 次方。因此，若使用了自定义初始化容量的构造方法来构造 HashMap 时（public HashMap(int initialCapacity)），需要计算一个 >= initialCapacity 的 2 的 n 次方数，为最终的 HashMap 的容量大小。

~~~java
/**
 * Returns a power of two size for the given target capacity.
 */
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    // === 下面说的’高位’指的是，二进制为 1 的 bit。比如，n 的二进制表示中第一个为 1 的 bit，为第1高位。
    n |= n >>> 1;    // 将第2高位置1
    n |= n >>> 2;    // 将第3~4高位置1
    n |= n >>> 4;    // 将第5~8高位置1
    n |= n >>> 8;    // 将第9~16高位置1
    n |= n >>> 16;    // 将第17~32高位置1
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
~~~

算法的思路是从 capacity 二进制的高位->低位都置为1，然后将这个全是1的二进制+1，得到 2 的 n 次方值。
在最开始，将『n = cap - 1』是为了防止 cap 已经是2的幂了。如果 cap 已经是2的幂， 又没有执行这个减1操作，则执行完后面的几条无符号右移操作之后，返回的 capacity 将是这个 cap 的2倍。

示例：
<img src="https://p.ipic.vip/6gds8n.png" style="display: block;margin-left: auto;margin-right: auto;width: 70%;"/>

#### 定位元素在 table 数组中的索引位置
索引的定位分为两步：
① 计算 key 的’散列值’
② 对得到的’散列值’进行取模，得到索引值

① 计算 key 的’散列值’  
~~~java
/**
 * Computes key.hashCode() and spreads (XORs) higher bits of hash
 * to lower.  Because the table uses power-of-two masking, sets of
 * hashes that vary only in bits above the current mask will
 * always collide. (Among known examples are sets of Float keys
 * holding consecutive whole numbers in small tables.)  So we
 * apply a transform that spreads the impact of higher bits
 * downward. There is a tradeoff between speed, utility, and
 * quality of bit-spreading. Because many common sets of hashes
 * are already reasonably distributed (so don't benefit from
 * spreading), and because we use trees to handle large sets of
 * collisions in bins, we just XOR some shifted bits in the
 * cheapest possible way to reduce systematic lossage, as well as
 * to incorporate impact of the highest bits that would otherwise
 * never be used in index calculations because of table bounds.
 */
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
~~~

算法的思路是，key的’hash值’高16位不变，低16位与高16位异或作为key的最终’散列值’。（h >>> 16，表示无符号右移16位，高位补0，任何数跟0异或都是其本身，因此key的hash值高16位不变。）

<img src="https://p.ipic.vip/lpad8y.png" style="display: block;margin-left: auto;margin-right: auto;width: 70%;"/>

> Q：为什么要这么干呢？  
> A：这第②步的取模操作有关：『index = (size - 1) & hash』

<br>

② 取模操作  
此处的取模操作，并非’传统’的’%’操作：『index = hash % size』；
因为 HashMap 的容量总是 2 的 n 次方。因此，HashMap使用『index = (size - 1) & hash』方式来实现取模操作（’size-1’的二进制的每一个bit都为1。并且’&’的操作效率要远高于’%’操作）。
但是，这也导致了一个问题，当 size 不是非常大的情况下，只有’散列值’的低位会参与到’取模’的计算中。因为，size 不够大的话，size二进制的高位全为 0 了。

举例：假设table.length=2^4=16

<img src="https://p.ipic.vip/5nst96.png" style="display: block;margin-left: auto;margin-right: auto;width: 70%;"/>
由上图可以看到，只有hash值的低4位参与了运算。   

这样做很容易产生碰撞。设计者权衡了speed, utility, and quality，将高16位与低16位异或来减少这种影响。设计者考虑到现在的hashCode分布的已经很不错了，而且当发生较大碰撞时也用树形存储降低了冲突。仅仅异或一下，既减少了系统的开销，也不会造成因为高位没有参与索引的计算(table长度比较小时)，从而引起的碰撞。

<br>

#### 元素的插入
~~~java
/**
 * Associates the specified value with the specified key in this map.
 * If the map previously contained a mapping for the key, the old
 * value is replaced.
 *
 * @param key key with which the specified value is to be associated
 * @param value value to be associated with the specified key
 * @return the previous value associated with <tt>key</tt>, or
 *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
 *         (A <tt>null</tt> return can also indicate that the map
 *         previously associated <tt>null</tt> with <tt>key</tt>.)
 */
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}


/**
 * Implements Map.put and related methods
 *
 * @param hash hash for key
 * @param key the key
 * @param value the value to put
 * @param onlyIfAbsent if true, don't change existing value
 * @param evict if false, the table is in creation mode.
 * @return previous value, or null if none
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
~~~
a）首先判断table是否为null或者长度为0，如果是，那么调用方法resize来初始化table。resize这个方法用来对HashMap的table数组扩容，它将发生在初始化table以及table中的记录数量达到阈值之后。
b）计算 key 的’散列值’，并得到’key’在table中的index。如果table index 上为 null，则直接调用newNode来创建一个新的链表节点，然后放在table的index位置上，此时表明没有哈希冲突。
c）如果 table 的 index 位置不为空，那么说明造成了哈希冲突，这时候如果key和index位置上的node的key相等（此时的 node 要么为’链表’的表头节点，要么为’红黑树’的根节点），则直接覆盖，否则继续下面流程。
d）如果index位置上的节点TreeNode，如果是，那么说明此时的index位置上是一颗红黑树，需要调用putTreeVal方法来将这新的记录插入到红黑树中去。
‘红黑树’的操作逻辑思路间：[图解“红黑树”]()
否则走下面的逻辑。
e）如果index位置上的节点类型不是TreeNode，那么说明此位置上的哈希冲突还没有达到阈值，还是一个链表结构。
那么就遍历链表，如果在链表中找到了这个key，则说明节点已经存，修改该节点的value为最新值即可；若遍历完列表依旧没有找到这个key，则说明是新节点需要插入，那么就调用newNode来创建一个新的链表节点，并插入链表尾部。
最后，如果插入了新的节点之后达到了阈值，那么就需要调用方法treeifyBin来讲链表转化为红黑树。
f）在插入完成之后，HashMap中的节点数量是否达到了设置的阈值（threshold），如果达到了，那么就需要调用方法resize来扩容。
<br>

#### 扩容操作
~~~java
/**
 * Initializes or doubles table size.  If null, allocates in
 * accord with initial capacity target held in field threshold.
 * Otherwise, because we are using power-of-two expansion, the
 * elements from each bin must either stay at same index, or move
 * with a power of two offset in the new table.
 *
 * @return the table
 */
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
~~~
实现的思路是：
① 创建一个容量为原数组大小2倍的新table数组
② 将原table数组中的元素，从新哈希落到新table数组中，然后将老的table数组释放掉。

由于数组的长度总是为 2 的 n 次方。因此，在扩容后，元素的新索引，要么和原先的old_index 值一样，要么为 old_index + old_table_length（即，new_index）。这是因为，我们计算索引的方法为:(hashCode & (length - 1))，而扩容将导致(length - 1)会新增一个1，也就是说，hashCode将会多一位来做判断，如果这个需要新判断的位置上为0，那么index不变，否则变为需要迁移到(oldIndex + oldCap)这个位置上去，下面举个例子吧：
~~~
还是上面的两个元素A和B，哈希值分别为3和47，在table长度为4的情况下，因为(3) = (11)，所以A和B会有两位参与运算来
获得index，A和B的二进制分别为：

3 ： 11
47： 101111

在table的length为4的前提下：

3-> 11 & 11 = 3
47-> 000011 & 101111 = 3

在扩容后，length变为8：
3-> 011 & 111 = 3
47-> 10111 & 00111 = 7

对于3来说，新增的参与运算的位为0，所以index不变，而对于47来说，新增的参与运算的位为1，所以
index需要变为(index + oldCap)
~~~

<span style='color:green'>从而也能得到一个结论，在扩容后，原来桶中’链表’的长度将减半，或’红黑树’的高度将减半！</span>

<br>

#### 元素的查找
~~~java
/**
 * Returns the value to which the specified key is mapped,
 * or {@code null} if this map contains no mapping for the key.
 *
 * <p>More formally, if this map contains a mapping from a key
 * {@code k} to a value {@code v} such that {@code (key==null ? k==null :
 * key.equals(k))}, then this method returns {@code v}; otherwise
 * it returns {@code null}.  (There can be at most one such mapping.)
 *
 * <p>A return value of {@code null} does not <i>necessarily</i>
 * indicate that the map contains no mapping for the key; it's also
 * possible that the map explicitly maps the key to {@code null}.
 * The {@link #containsKey containsKey} operation may be used to
 * distinguish these two cases.
 *
 * @see #put(Object, Object)
 */
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}


/**
 * Implements Map.get and related methods
 *
 * @param hash hash for key
 * @param key the key
 * @return the node, or null if none
 */
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
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
~~~

方法思路是：
a）先根据 key 定位到节点会在table数组中的索引位置。
b）再在相应的桶中查找节点。如果桶中是’链表’的话，就顺序查找；如果桶中是’红黑树’的话，就遍历’红黑树’进行查找
c）在上面的查找过程中，如果没找到，则说明节点不存在。返回 null。否则，返回节点的值 value。

> 注意：由于节点的 value 可以设置为 null。因此，该方法返回 null 时，不能说明是节点不存在。也就是我们不能通过 HashMap 的 get 方法的返回值来判断节点是否存在，而应该使用 containsKey 方法来判断。

<br>

#### 元素的删除
~~~java
/**
 * Removes the mapping for the specified key from this map if present.
 *
 * @param  key key whose mapping is to be removed from the map
 * @return the previous value associated with <tt>key</tt>, or
 *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
 *         (A <tt>null</tt> return can also indicate that the map
 *         previously associated <tt>null</tt> with <tt>key</tt>.)
 */
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}


/**
 * Implements Map.remove and related methods
 *
 * @param hash hash for key
 * @param key the key
 * @param value the value to match if matchValue, else ignored
 * @param matchValue if true only remove if value is equal
 * @param movable if false do not move other nodes while removing
 * @return the node, or null if none
 */
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
~~~
方法的思路是：
a）先查找到待删除的节点。如果节点不存在，则直接返回。否则，继续下面的步骤
b）如果待删除节点，在链表中，则将其从链表中移除；如果待删除的节点在’红黑树’中，则将其从树中移除，如若必要对’红黑树’进行重新平衡。
关于’红黑树’的节点删除操作原理，可见：[图解“红黑树”]()

<br>

## HashMap 线程不安全

### 3.1 ConcurrentModificationException 异常

当一个线程在遍历 HashMap 的过程，该 HashMap 被其他线程修改了，那边遍历的过程会被异常终止，并抛出 ‘ConcurrentModificationException’ 异常。


### 3.2 JDK8 之前 HashMap put 操作导致 CPU 100% 情景

CPU 100% 实际是因为在`多线程`写操作使map容量达到阈值因此需要`扩容操作`，该操作可能`导致Entry闭环`，从而导致迭代操作时数据存在`“取不完”的假象`。

> 解决方法：
> 其实并没有什么所谓的解决方法，这是对HashMap的一个错误使用方式造成的问题。如果在多线程下我们应该是用 ConcurrentHashMap 而不再是使用 HashMap 了。

