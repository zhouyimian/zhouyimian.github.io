---
layout:     post
title:      JDK源码阅读—HashMap（1）
subtitle:   
date:       2019-6-29
author:     BY KiloMeter
header-img: img/2019-6-29-JDK源码阅读—HashMap（1）/62.jpg
catalog: true
tags:
    - 源码阅读
---

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    }
```

HashMap实现了AbstractMap，可以看一下之前写过的 [JDK源码阅读—AbstractMap](https://zhouyimian.github.io/2019/06/22/JDK源码阅读-AbstractMap/)

## 特殊常量

```java
//默认初始化容量为16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
//最大容量为2^30
static final int MAXIMUM_CAPACITY = 1 << 30;
//负载因子为0.75，负载因子指的是，键值对的数量/数组的长度，如果超出负载因子，则需要对数组进行扩容
static final float DEFAULT_LOAD_FACTOR = 0.75f;
//由链表转换成树的阈值
static final int TREEIFY_THRESHOLD = 8;
//resize后由树转换成链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;
//其实说链表长度大于8就转换成红黑树这个说法是不准确的，准确来说，当链表长度大于8时，优先进行扩容而不是
//转换成红黑树，只有当数组长度大于MIN_TREEIFY_CAPACITY后才会转换成红黑树
static final int MIN_TREEIFY_CAPACITY = 64;
```

## 节点结构

```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;  //从next可以看到每个节点都是一个链表

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
transient Node<K,V>[] table;   //HashMap底层为数组结构
transient Set<Map.Entry<K,V>> entrySet;
transient int size;
transient int modCount;//HashMap也是线程不安全的，这里也有和ArrayList中一样作用的modCount变量
int threshold;  //下次resize的元素数量(即capacity * load factor)
final float loadFactor;
```



## 构造方法

```java
public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
//由于HasMap的capacity都是2的次幂，在初始化时如果制定了initialCapacity，那么该方法会找到大于等于initialCapacity的最小的2的幂
static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

## 插入数据

```java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) {
            //如果table还未初始化
            if (table == null) { // pre-size
                //计算下一次resize的阈值
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                //如果插入数据后的长度大于当前的resize阈值，那么需要进行扩容
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            else if (s > threshold)
                resize();
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //如果HashMap是刚刚创建，则会在这里调用resize()方法进行初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //(n - 1) & hash是要插入的数组位置下标，如果为null的话，直接插入
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            //如果根节点和要插入节点的key完全相同，则直接覆盖
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //如果当前位置的根节点是红黑树的话
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //当前位置的根节点是链表
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        //如果链表中所有节点的key都与要插入的节点的key不相同，则插入数据
                        p.next = newNode(hash, key, value, null);
                        //如果链表长度超出了8，则转换为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果链表中某个节点的key和要插入节点的key相同，直接退出
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //如果e!=null，意味着这个位置的链表或者红黑树中存在key相同的节点，此时需要对旧的节点的value值进行替换
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //代码跑到这里意味着插入了数据(因为如果是替换的话，在前面e!=null的时候就已经return了)如果插入后的节点数量大于负载树，则需要扩容。
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
//将链表转换成红黑树
final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        //如果数组长度还未达到MIN_TREEIFY_CAPACITY，则会优先进行扩容而不是将链表转换成红黑树
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            //该循环将单链表的结点替换成TreeNode树结点，并构建双向链表，为构建红黑树做准备
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                //将双向链表转换为红黑树
                hd.treeify(tab);
        }
    }
//这个方法发生在table初始化，或者table中的元素数量超出threshold的时候。
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        //如果此时的HashMap中已经有数据了
        if (oldCap > 0) {
//如果原来HashMap的capacity已经超过了MAXIMUM_CAPACITY，那么已经无法进行扩容了，只能够增加扩容阈值
//因为触发resize()方法是因为元素的数量超出了扩容阈值(loadfactory*capacity)，在这里，直接修改扩容阈值
//因此这个时候，扩容阈值不再是loadfactory*capacity了，而是Integer.MAX_VALUE
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            
            //这里在修改扩容后再次扩容的阈值，如果旧容量*2<MAXIMUM_CAPACITY并且旧容量>=默认容量16，那么再次扩容后的阈值为旧的扩容阈值*2
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        //这里oldCap == 0，即HashMap还没有数据的情况
        //如果在初始化时指定了initialcapacity，那么用threshold作为table的实际大小
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        //这里oldCap == 0且oldThr == 0，即刚开始初始化且没有指定initialcapacity
        //那么会设置默认数组默认大小为16，负载为DEFAULT_LOAD_FACTOR*DEFAULT_INITIAL_CAPACITY
        else {
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        //计算指定了initialCapacity情况下的新的threshold，即上面else if (oldThr > 0)这种情况
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        //把原来的所有元素全部遍历存储到新的数组中
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    //如果链表只有一个元素
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    //如果此时的节点后面已经是红黑树而不是链表
                    else if (e instanceof TreeNode)
                        //split这个关于红黑树的方法放在下面进行详细解释
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //节点后面是链表
                    else {
                        //这一部分比较巧妙，放在下面详细将
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
```

单独看下链表重新打散这段精巧的代码(以下内容来自 [深入理解HashMap(四): 关键源码逐行分析之resize扩容](https://segmentfault.com/a/1190000015812438))

```java
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
```

首先我们看源码时要抓住一个大框架, 不要被它复杂的流程唬住, 我们一段一段来看:

**第一段**

```java
Node<K,V> loHead = null, loTail = null;
Node<K,V> hiHead = null, hiTail = null;
```

上面这段定义了四个Node的引用, 从变量命名上,我们初步猜测, 这里定义了两个链表, 我们把它称为 `lo链表` 和 `hi链表`, `loHead` 和 `loTail` 分别指向 `lo链表`的头节点和尾节点, `hiHead` 和 `hiTail`以此类推.

**第二段**

```java
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
```

上面这段是一个do-while循环, 我们先从中提取出主要框架:

```java
do {
    next = e.next;
    ...
} while ((e = next) != null);
```

从上面的框架上来看, 就是在按顺序遍历该存储桶位置上的链表中的节点.

我们再看`if-else` 语句的内容:

```java
// 插入lo链表
if (loTail == null)
    loHead = e;
else
    loTail.next = e;
loTail = e;

// 插入hi链表
if (hiTail == null)
    hiHead = e;
else
    hiTail.next = e;
hiTail = e;
```

上面结构类似的两段看上去就是一个将节点e插入链表的动作.

最后再加上 `if` 块, 则上面这段的目的就很清晰了:

> 我们首先准备了两个链表 `lo` 和 `hi`, 然后我们顺序遍历该存储桶上的链表的每个节点, 如果 `(e.hash & oldCap) == 0`, 我们就将节点放入`lo`链表, 否则, 放入`hi`链表.

**第三段**

第二段弄明白之后, 我们再来看第三段:

```java
if (loTail != null) {
    loTail.next = null;
    newTab[j] = loHead;
}
if (hiTail != null) {
    hiTail.next = null;
    newTab[j + oldCap] = hiHead;
}
```

这一段看上去就很简单了:

> 如果lo链表非空, 我们就把整个lo链表放到新table的`j`位置上 
> 如果hi链表非空, 我们就把整个hi链表放到新table的`j+oldCap`位置上

综上我们知道, 这段代码的意义就是将原来的链表拆分成两个链表, 并将这两个链表分别放到新的table的 `j` 位置和 `j+oldCap` 上, `j`位置就是原链表在原table中的位置, 拆分的标准就是:

```java
(e.hash & oldCap) == 0
```

**关于 `(e.hash & oldCap) == 0` `j` 以及 `j+oldCap`**

上面我们已经弄懂了链表拆分的代码, 但是这个拆分条件看上去很奇怪, 这里我们来稍微解释一下:

首先我们要明确三点:

1. oldCap一定是2的整数次幂, 这里假设是2^m
2. newCap是oldCap的两倍, 则会是2^(m+1)
3. hash对数组大小取模`(n - 1) & hash` 其实就是取hash的低`m`位

例如: 
我们假设 oldCap = 16, 即 2^4, 
16 - 1 = 15, 二进制表示为 `0000 0000 0000 0000 0000 0000 0000 1111` 
可见除了低4位, 其他位置都是0（简洁起见，高位的0后面就不写了）, 则 `(16-1) & hash` 自然就是取hash值的低4位,我们假设它为 `abcd`.

以此类推, 当我们将oldCap扩大两倍后, 新的index的位置就变成了 `(32-1) & hash`, 其实就是取 hash值的低5位. 那么对于同一个Node, 低5位的值无外乎下面两种情况:

```
0abcd
1abcd
```

其中, `0abcd`与原来的index值一致, 而`1abcd` = `0abcd + 10000` = `0abcd + oldCap`

故虽然数组大小扩大了一倍，但是同一个`key`在新旧table中对应的index却存在一定联系： 要么一致，要么相差一个 `oldCap`。

而新旧index是否一致就体现在hash值的第4位(我们把最低为称作第0位), 怎么拿到这一位的值呢, 只要:

```
hash & 0000 0000 0000 0000 0000 0000 0001 0000
```

上式就等效于

```java
hash & oldCap
```

故得出结论:

> 如果 `(e.hash & oldCap) == 0` 则该节点在新表的下标位置与旧表一致都为 `j` 
> 如果 `(e.hash & oldCap) == 1` 则该节点在新表的下标位置 `j + oldCap`

根据这个条件, 我们将原位置的链表拆分成两个链表, 然后一次性将整个链表放到新的Table对应的位置上.



下面这段代码是在resize()时，如果节点后面是红黑树的情况。

```java
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
            TreeNode<K,V> b = this;
            //这里的处理和链表的处理基本是一样的，这里的lo和hi是先以双向链表的形式存放数据
            TreeNode<K,V> loHead = null, loTail = null;
            TreeNode<K,V> hiHead = null, hiTail = null;
            //lc和hc用来记录新位置的元素数量，用于最后是要以红黑树还是以链表的形式进行存储
            int lc = 0, hc = 0;
            for (TreeNode<K,V> e = b, next; e != null; e = next) {
                next = (TreeNode<K,V>)e.next;
                e.next = null;
                if ((e.hash & bit) == 0) {
                    if ((e.prev = loTail) == null)
                        loHead = e;
                    else
                        loTail.next = e;
                    loTail = e;
                    ++lc;
                }
                else {
                    if ((e.prev = hiTail) == null)
                        hiHead = e;
                    else
                        hiTail.next = e;
                    hiTail = e;
                    ++hc;
                }
            }

            if (loHead != null) {
                //如果长度小于UNTREEIFY_THRESHOLD(这个值为6，是红黑树重新转换成链表的阈值)
                if (lc <= UNTREEIFY_THRESHOLD)
                    tab[index] = loHead.untreeify(map);
                else {
                    tab[index] = loHead;
                    if (hiHead != null) // (else is already treeified)
                        loHead.treeify(tab);
                }
            }
            if (hiHead != null) {
                if (hc <= UNTREEIFY_THRESHOLD)
                    tab[index + bit] = hiHead.untreeify(map);
                else {
                    tab[index + bit] = hiHead;
                    if (loHead != null)
                        hiHead.treeify(tab);
                }
            }
        }
//把双向链表改为单向链表
final Node<K,V> untreeify(HashMap<K,V> map) {
            Node<K,V> hd = null, tl = null;
            for (Node<K,V> q = this; q != null; q = q.next) {
                Node<K,V> p = map.replacementNode(q, null);
                if (tl == null)
                    hd = p;
                else
                    tl.next = p;
                tl = p;
            }
            return hd;
        }
Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
        return new Node<>(p.hash, p.key, p.value, next);
    }
//把双向链表改为红黑树
//这段以后再详细分析 红黑树相关的还不是很清楚
final void treeify(Node<K,V>[] tab) {
            TreeNode<K,V> root = null;
            for (TreeNode<K,V> x = this, next; x != null; x = next) {
                next = (TreeNode<K,V>)x.next;
                x.left = x.right = null;
                if (root == null) {
                    x.parent = null;
                    x.red = false;
                    root = x;
                }
                else {
                    K k = x.key;
                    int h = x.hash;
                    Class<?> kc = null;
                    for (TreeNode<K,V> p = root;;) {
                        int dir, ph;
                        K pk = p.key;
                        if ((ph = p.hash) > h)
                            dir = -1;
                        else if (ph < h)
                            dir = 1;
                        else if ((kc == null &&
                                  (kc = comparableClassFor(k)) == null) ||
                                 (dir = compareComparables(kc, k, pk)) == 0)
                            dir = tieBreakOrder(k, pk);

                        TreeNode<K,V> xp = p;
                        if ((p = (dir <= 0) ? p.left : p.right) == null) {
                            x.parent = xp;
                            if (dir <= 0)
                                xp.left = x;
                            else
                                xp.right = x;
                            root = balanceInsertion(root, x);
                            break;
                        }
                    }
                }
            }
            moveRootToFront(tab, root);
        }
```



## 移除数据

```java
public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }
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
public void clear() {
        Node<K,V>[] tab;
        modCount++;
        if ((tab = table) != null && size > 0) {
            size = 0;
            for (int i = 0; i < tab.length; ++i)
                tab[i] = null;
        }
    }
```

